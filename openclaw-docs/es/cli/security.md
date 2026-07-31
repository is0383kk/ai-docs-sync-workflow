---
read_when:
    - Se desea ejecutar una auditoría de seguridad rápida de la configuración y el estado
    - Se desea aplicar sugerencias de «corrección» seguras (permisos, valores predeterminados más estrictos)
summary: Referencia de la CLI para `openclaw security` (auditar y corregir errores comunes de seguridad)
title: Seguridad
x-i18n:
    generated_at: "2026-07-26T04:36:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b5f9ea5cb746bfd29ff4d096062e81595abe99a883fc3b1113b45a3527d42d9
    source_path: cli/security.md
    workflow: 16
---

# `openclaw security`

Herramientas de seguridad: auditoría y correcciones seguras opcionales. Véase también: [Seguridad](/es/gateway/security).

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --auth password --password <password>
openclaw security audit --fix
openclaw security audit --json
```

## Modos de auditoría

La ejecución simple de `security audit` permanece en la ruta de configuración/sistema de archivos de solo lectura y sin inicialización: no detecta los recopiladores de seguridad del entorno de ejecución de los plugins, por lo que las auditorías rutinarias no cargan el entorno de ejecución de cada plugin instalado. `--deep` añade sondeos en vivo del Gateway, sujetos al mejor esfuerzo, y recopiladores de auditoría de seguridad pertenecientes a los plugins (los llamadores internos explícitos también pueden optar por esos recopiladores cuando ya disponen de un ámbito de entorno de ejecución adecuado).

Si la autenticación por contraseña del Gateway se proporciona únicamente durante el inicio, pase el mismo valor con `--auth password --password <password>` para que la auditoría pueda comprobarlo con respecto a `hooks.token`.

## Qué comprueba

**Modelo de mensajes directos/confianza**

- Advierte cuando varios remitentes de mensajes directos comparten la sesión principal y recomienda el modo seguro para mensajes directos: `session.dmScope="per-channel-peer"` (o `per-account-channel-peer` para canales con varias cuentas) en bandejas de entrada compartidas. Esto refuerza la seguridad de la cooperación y las bandejas compartidas; no proporciona aislamiento entre operadores que no confían entre sí. Separe los límites de confianza mediante gateways independientes (o usuarios/hosts del sistema operativo independientes).
- Emite `security.trust_model.multi_user_heuristic` cuando la configuración indica una probable entrada de varios usuarios (por ejemplo, una política abierta para mensajes directos/grupos, destinos de grupos configurados o reglas de remitentes con comodines). El modelo de confianza predeterminado de OpenClaw es el de un asistente personal (un solo operador), no el aislamiento multiinquilino frente a actores hostiles. En configuraciones compartidas intencionalmente entre varios usuarios: ejecute todas las sesiones en un entorno aislado, limite el acceso al sistema de archivos al espacio de trabajo y mantenga las identidades o credenciales personales/privadas fuera de ese entorno de ejecución.
- Advierte cuando se utilizan modelos pequeños (parámetros `<=300B`) sin aislamiento y con las herramientas web/del navegador habilitadas.

**Webhook/hooks**

Durante el inicio se registra una advertencia de seguridad no fatal, y la auditoría señala la reutilización de `hooks.token` en valores activos de autenticación mediante secreto compartido del Gateway (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN`, `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`). También advierte cuando:

- `hooks.token` es corto
- `hooks.path="/"`
- `hooks.defaultSessionKey` no está establecido
- `hooks.allowedAgentIds` no tiene restricciones
- están habilitadas las sustituciones de `sessionKey` de las solicitudes
- están habilitadas las sustituciones sin `hooks.allowedSessionKeyPrefixes`

Ejecute `openclaw doctor --fix` para rotar un `hooks.token` persistente y reutilizado; después, actualice los remitentes externos de hooks para que utilicen el nuevo token.

**Entorno aislado/herramientas**

- Advierte cuando se han configurado los ajustes de Docker del entorno aislado mientras el modo de aislamiento está desactivado.
- Advierte cuando `gateway.nodes.commands.deny` utiliza entradas desconocidas o similares a patrones que no surten efecto (la coincidencia solo se realiza con el nombre exacto del comando del Node, no mediante el filtrado del texto del shell).
- Advierte cuando `gateway.nodes.commands.allow` habilita explícitamente comandos peligrosos del Node.
- Advierte cuando los perfiles de herramientas de los agentes sustituyen el valor global de `tools.profile="minimal"`.
- Advierte cuando las herramientas de escritura/edición están deshabilitadas, pero `exec` sigue disponible sin un límite restrictivo del sistema de archivos del entorno aislado.
- Advierte cuando los mensajes directos o grupos abiertos exponen herramientas del entorno de ejecución/sistema de archivos sin protecciones de aislamiento/espacio de trabajo.
- Advierte cuando las herramientas de plugins instalados pueden resultar accesibles con una política permisiva de herramientas.

**Navegador del entorno aislado**

- Advierte cuando el navegador del entorno aislado utiliza la red `bridge` de Docker sin `sandbox.browser.cdpSourceRange`.
- Señala modos peligrosos de red de Docker para el entorno aislado, incluidas las uniones a los espacios de nombres `host` y `container:*`.
- Advierte cuando los contenedores de Docker existentes del navegador del entorno aislado tienen etiquetas hash ausentes u obsoletas (por ejemplo, contenedores anteriores a la migración sin `openclaw.browserConfigEpoch`) y recomienda `openclaw sandbox recreate --browser --all`.

**Red/detección**

- Señala `gateway.allowRealIpFallback=true` (riesgo de suplantación de encabezados si los proxies están mal configurados).
- Señala `discovery.mdns.mode="full"` (filtración de metadatos mediante registros TXT de mDNS).
- Advierte cuando `gateway.auth.mode="none"` deja accesibles las API HTTP del Gateway sin un secreto compartido (`/tools/invoke` y cualquier endpoint `/v1/*` habilitado).

**Plugins/canales**

- Advierte cuando los registros de instalación de plugins/hooks basados en npm no están fijados a una versión, carecen de metadatos de integridad o difieren de las versiones de los paquetes instalados actualmente.
- Advierte cuando las listas de permitidos de los canales dependen de nombres/correos electrónicos/etiquetas modificables en lugar de identificadores estables (en los ámbitos de Discord, Slack, Google Chat, Microsoft Teams, Mattermost e IRC donde corresponda).

Los ajustes con los prefijos `dangerous`/`dangerously` son sustituciones explícitas de emergencia del operador; habilitar una de ellas no constituye, por sí solo, un informe de vulnerabilidad de seguridad. Para consultar el inventario completo de parámetros peligrosos, véase «Resumen de indicadores inseguros o peligrosos» en [Seguridad](/es/gateway/security).

## Comportamiento de SecretRef

`security audit` resuelve las SecretRefs compatibles en modo de solo lectura para las rutas que tiene como objetivo. Si una SecretRef no está disponible en la ruta del comando actual, la auditoría continúa e informa de `secretDiagnostics` en lugar de bloquearse. `--token` y `--password` solo sustituyen la autenticación del sondeo profundo durante esa invocación del comando; no reescriben la configuración ni las asignaciones de SecretRef.

## Supresiones

Acepte los hallazgos persistentes intencionales mediante `security.audit.suppressions`. Cada supresión coincide con un `checkId` exacto y puede limitarse mediante subcadenas `titleIncludes` y/o `detailIncludes` que no distinguen entre mayúsculas y minúsculas:

```json
{
  "security": {
    "audit": {
      "suppressions": [
        {
          "checkId": "plugins.tools_reachable_permissive_policy",
          "detailIncludes": "Enabled extension plugins: gbrain",
          "reason": "trusted local operator plugin"
        }
      ]
    }
  }
}
```

Los hallazgos suprimidos se eliminan de las listas activas `summary` y `findings`. La salida JSON los conserva en `suppressedFindings` para permitir su auditoría. Cuando se configuran supresiones, la salida activa también conserva un hallazgo informativo `security.audit.suppressions.active` que no se puede suprimir, de modo que los lectores puedan saber que la auditoría se filtró. Los indicadores de configuración peligrosos se emiten como un hallazgo por indicador; por tanto, aceptar un indicador peligroso no oculta otros indicadores habilitados que compartan el mismo checkId de `config.insecure_or_dangerous_flags`.

Dado que las supresiones pueden ocultar riesgos persistentes, añadirlas o eliminarlas mediante comandos de shell ejecutados por agentes requiere aprobación para la ejecución, salvo que esta ya se esté realizando con `security="full"` y `ask="off"` para automatización local de confianza.

## Salida JSON

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

Con `--fix --json`, la salida incluye tanto las acciones correctivas como el informe final:

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## Qué cambia `--fix`

Aplica correcciones seguras y deterministas:

- cambia los valores comunes de `groupPolicy="open"` a `groupPolicy="allowlist"` (incluidas las variantes de cuenta en los canales compatibles)
- cuando la política de grupos de WhatsApp cambia a `allowlist`, inicializa `groupAllowFrom` a partir del archivo `allowFrom` almacenado si dicha lista existe y la configuración aún no define `allowFrom`
- cambia `logging.redactSensitive` de `"off"` a `"tools"`
- restringe los permisos del estado/la configuración y de los archivos confidenciales habituales (`credentials/*.json`, `auth-profiles.json`, `openclaw-agent.sqlite` y artefactos de sesión heredados)
- también restringe los permisos de los archivos de inclusión de configuración referenciados desde `openclaw.json`
- utiliza `chmod` en hosts POSIX y restablecimientos de `icacls` en Windows

`--fix` **no**:

- rota tokens/contraseñas/claves de API
- deshabilita herramientas (`gateway`, `cron`, `exec`, etc.)
- cambia las opciones de enlace/autenticación/exposición de red del Gateway
- elimina ni reescribe plugins/Skills

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Auditoría de seguridad](/es/gateway/security)
