---
read_when:
    - El centro de solución de problemas le remitió aquí para realizar un diagnóstico más profundo
    - Se necesitan secciones estables de procedimientos operativos basadas en síntomas, con comandos exactos
sidebarTitle: Troubleshooting
summary: Guía exhaustiva de solución de problemas para el Gateway, los canales, la automatización, los nodos y el navegador
title: Solución de problemas
x-i18n:
    generated_at: "2026-07-26T04:42:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4bb1e061dbf2767118c24ad1ca2d2d1f7eeeff88e18ed0e6111aebe1cc99a26
    source_path: gateway/troubleshooting.md
    workflow: 16
---

Este es el manual operativo detallado. Empiece primero por [/help/troubleshooting](/es/help/troubleshooting) para seguir el flujo de triaje rápido.

## Secuencia de comandos

Ejecútelos en este orden:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Indicadores de funcionamiento correcto:

- `openclaw gateway status` muestra `Runtime: running`, `Connectivity probe: ok` y una línea `Capability: ...`.
- `openclaw doctor` no informa de problemas de configuración o servicio que impidan continuar.
- `openclaw channels status --probe` muestra el estado activo del transporte por cuenta y, cuando se admite, `works` o `audit ok`.

## Después de una actualización

Utilice esta sección cuando finalice una actualización, pero el Gateway esté inactivo, los canales estén vacíos o las llamadas a modelos fallen con errores 401.

```bash
openclaw status --all
openclaw update status --json
openclaw gateway status --deep
openclaw doctor --fix
openclaw gateway restart
```

Compruebe lo siguiente:

- `Update restart` en `openclaw status` / `openclaw status --all`. Las transferencias pendientes o fallidas incluyen el siguiente comando que debe ejecutarse.
- `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` en Canales: la configuración del canal todavía existe, pero el registro del plugin falló antes de que pudiera cargarse el canal.
- Errores 401 del proveedor después de volver a autenticarse: `openclaw doctor --fix` busca copias obsoletas de autenticación OAuth por agente y elimina las antiguas para que todos los agentes resuelvan el perfil compartido actual.

## Instalaciones divergentes y protección frente a configuraciones más recientes

Utilice esta sección cuando un servicio del Gateway se detenga inesperadamente después de una actualización o cuando los registros indiquen que un binario `openclaw` es anterior a la versión que escribió por última vez `openclaw.json`.

OpenClaw marca las escrituras de configuración con `meta.lastTouchedVersion`. Los comandos de solo lectura pueden inspeccionar una configuración escrita por una versión más reciente de OpenClaw, pero las mutaciones de procesos y servicios se niegan a ejecutarse desde un binario anterior. Acciones bloqueadas: iniciar, detener, reiniciar o desinstalar el servicio del Gateway; forzar la reinstalación del servicio; iniciar el Gateway en modo de servicio; y limpiar el puerto `gateway --force`.

```bash
which openclaw
openclaw --version
openclaw gateway status --deep
openclaw config get meta.lastTouchedVersion
```

<Steps>
  <Step title="Corregir PATH">
    Corrija `PATH` para que `openclaw` resuelva la instalación más reciente y vuelva a ejecutar la acción.
  </Step>
  <Step title="Reinstalar el servicio del Gateway">
    Reinstale el servicio del Gateway previsto desde la instalación más reciente:

    ```bash
    openclaw gateway install --force
    openclaw gateway restart
    ```

  </Step>
  <Step title="Eliminar envoltorios obsoletos">
    Elimine los paquetes del sistema obsoletos o las entradas de envoltorios antiguos que todavía apunten a un binario `openclaw` anterior.
  </Step>
</Steps>

<Warning>
Solo para una reversión intencionada o una recuperación de emergencia, establezca `OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1` para ese único comando. Déjelo sin establecer durante el funcionamiento normal.
</Warning>

## Incompatibilidad de protocolo después de una reversión

Utilice esta sección cuando los registros sigan mostrando `protocol mismatch` después de una degradación o reversión. Se está ejecutando un Gateway anterior, pero un proceso cliente local más reciente sigue intentando reconectarse con un intervalo de protocolo que el Gateway anterior no admite.

```bash
openclaw --version
which -a openclaw
openclaw gateway status --deep
openclaw doctor --deep
openclaw logs --follow
```

Compruebe lo siguiente:

- `protocol mismatch ... client=... v<version> min=<n> max=<n> expected=<n>` en los registros del Gateway.
- `Established clients:` en `openclaw gateway status --deep` o `Gateway clients` en `openclaw doctor --deep`: clientes TCP activos conectados al puerto del Gateway, con PID y líneas de comandos cuando el sistema operativo lo permita.
- Un proceso cliente cuya línea de comandos apunte a la instalación o al envoltorio más reciente de OpenClaw desde el que se realizó la reversión.

Solución:

1. Detenga o reinicie el proceso cliente obsoleto de OpenClaw que muestra `gateway status --deep`.
2. Reinicie las aplicaciones o los envoltorios que incorporan OpenClaw: paneles locales, editores, asistentes de servidores de aplicaciones o shells `openclaw logs --follow` de larga duración.
3. Vuelva a ejecutar `openclaw gateway status --deep` o `openclaw doctor --deep` y confirme que el PID del cliente obsoleto ha desaparecido.

No haga que un Gateway anterior acepte un protocolo más reciente e incompatible. Los incrementos de versión del protocolo protegen el contrato de comunicación; la recuperación tras una reversión es un problema de limpieza de procesos y versiones.

## Enlace simbólico de una Skill omitido por escapar de la ruta

Utilice esta sección cuando los registros incluyan:

```text
Se omite la ruta de la skill que escapa de su raíz configurada: ... reason=symlink-escape
```

Cada raíz de Skills constituye un límite de contención. Un enlace simbólico en `~/.agents/skills`, `<workspace>/.agents/skills`, `<workspace>/skills` o `~/.openclaw/skills` se omite cuando su destino real se resuelve fuera de esa raíz, salvo que el destino sea explícitamente de confianza.

Inspeccione el enlace:

```bash
ls -l ~/.agents/skills/<name>
realpath ~/.agents/skills/<name>
openclaw config get skills.load
```

Si el destino es intencionado, configure tanto la raíz directa de la Skill como el destino permitido del enlace simbólico:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

Después, inicie una sesión nueva o espere a que se actualice el observador de Skills. Reinicie el Gateway si el proceso en ejecución es anterior al cambio de configuración.

No utilice destinos amplios como `~`, `/` o una carpeta completa de proyecto sincronizada. Mantenga `allowSymlinkTargets` limitado a la raíz real de Skills que contiene directorios `SKILL.md` de confianza.

Si la aplicación de Skill Workshop también debe escribir a través de esas rutas de Skills del espacio de trabajo enlazadas simbólicamente y de confianza, habilite `skills.workshop.allowSymlinkTargetWrites`. Manténgalo deshabilitado para las raíces compartidas de Skills de solo lectura.

Relacionado:

- [Configuración de Skills](/es/tools/skills-config#symlinked-skill-roots)
- [Ejemplos de configuración](/es/gateway/configuration-examples#symlinked-sibling-skill-repo)

## Anthropic 429: se requiere uso adicional para contextos largos

Utilice esta sección cuando los registros o errores incluyan: `HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

Compruebe lo siguiente:

- El modelo de Anthropic seleccionado es un modelo Claude 4.x de 1M con disponibilidad general (Opus 4.6/4.7/4.8, Sonnet 4.6), o la configuración del modelo todavía contiene el valor heredado `params.context1m: true`.
- La credencial actual de Anthropic no cumple los requisitos para usar contextos largos.
- Las solicitudes solo fallan en sesiones o ejecuciones de modelos largas que requieren la ruta de contexto de 1M.

Opciones de solución:

<Steps>
  <Step title="Usar una ventana de contexto estándar">
    Cambie a un modelo con una ventana estándar o elimine el valor heredado `context1m` de una
    configuración de modelo anterior que no tenga disponibilidad general para contextos de 1M.
  </Step>
  <Step title="Usar una credencial apta">
    Utilice una credencial de Anthropic apta para solicitudes de contexto largo o cambie a una clave de API de Anthropic.
  </Step>
  <Step title="Configurar modelos de reserva">
    Configure modelos de reserva para que las ejecuciones continúen cuando se rechacen las solicitudes de contexto largo de Anthropic.
  </Step>
</Steps>

Relacionado:

- [Anthropic](/es/providers/anthropic)
- [Uso y costes de tokens](/es/reference/token-use)
- [¿Por qué aparece el error HTTP 429 de Anthropic?](/es/help/faq-first-run#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## Respuestas 403 bloqueadas en el servicio ascendente

Utilice esta sección cuando un proveedor de LLM ascendente devuelva un `403` genérico, como `Your request was blocked`.

No presuponga que esto siempre se debe a un problema de configuración de OpenClaw. La respuesta puede proceder de una capa de seguridad ascendente, como una CDN, un WAF, una regla de gestión de bots o un proxy inverso situado delante de un endpoint compatible con OpenAI.

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
```

Compruebe lo siguiente:

- Varios modelos del mismo proveedor fallan de la misma manera.
- Aparece HTML o texto genérico de seguridad en lugar de un error normal de la API del proveedor.
- Hay eventos de seguridad del proveedor correspondientes a la misma hora de la solicitud.
- Una pequeña prueba directa con `curl` funciona, mientras que las solicitudes normales con la estructura del SDK fallan.

Corrija primero el filtrado del proveedor cuando las pruebas indiquen un bloqueo del WAF o la CDN. Es preferible utilizar una regla de permiso u omisión limitada específicamente a la ruta de la API que utiliza OpenClaw y evitar deshabilitar la protección de todo el sitio.

<Warning>
Una solicitud mínima correcta con `curl` no garantiza que las solicitudes reales con el formato del SDK atraviesen la misma capa de seguridad ascendente.
</Warning>

Relacionado:

- [Endpoints compatibles con OpenAI](/es/gateway/configuration-reference#openai-compatible-endpoints)
- [Configuración de proveedores](/es/providers)
- [Registros](/es/logging)

## El backend local compatible con OpenAI supera las pruebas directas, pero las ejecuciones del agente fallan

Utilice esta sección cuando:

- `curl ... /v1/models` funciona.
- Las llamadas directas pequeñas con `/v1/chat/completions` funcionan.
- Las ejecuciones de modelos de OpenClaw solo fallan durante los turnos normales del agente.

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"hi"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "hi" --json
openclaw logs --follow
```

Compruebe lo siguiente:

- Las llamadas directas pequeñas funcionan, pero las ejecuciones de OpenClaw solo fallan con prompts más grandes.
- Aparecen errores `model_not_found` o 404, aunque una solicitud directa con `/v1/chat/completions` funciona con el mismo identificador de modelo sin prefijo.
- El backend genera errores que indican que `messages[].content` esperaba una cadena.
- Aparecen advertencias intermitentes `incomplete turn detected ... stopReason=stop payloads=0` con un backend local compatible con OpenAI.
- El backend se bloquea únicamente con cantidades mayores de tokens del prompt o con los prompts completos del entorno de ejecución del agente.

<AccordionGroup>
  <Accordion title="Indicadores habituales">
    - `model_not_found` con un servidor local de estilo MLX/vLLM: compruebe que `baseUrl` incluya `/v1`, que `api` sea `"openai-completions"` para backends `/v1/chat/completions` y que `models.providers.<provider>.models[].id` sea el identificador local del proveedor sin prefijo. Selecciónelo una vez con el prefijo del proveedor, por ejemplo `mlx/mlx-community/Qwen3-30B-A3B-6bit`; mantenga la entrada del catálogo como `mlx-community/Qwen3-30B-A3B-6bit`.
    - `messages[...].content: invalid type: sequence, expected a string`: el backend rechaza las partes de contenido estructurado de Chat Completions. Solución: establezca `models.providers.<provider>.models[].compat.requiresStringContent: true`.
    - `validation.keys` o claves de mensaje permitidas como `["role","content"]`: el backend rechaza los metadatos de reproducción de estilo OpenAI en los mensajes de Chat Completions. Solución: establezca `models.providers.<provider>.models[].compat.strictMessageKeys: true`.
    - `incomplete turn detected ... stopReason=stop payloads=0`: el backend completó la solicitud de Chat Completions, pero no devolvió texto visible para el usuario en la respuesta del asistente durante ese turno. OpenClaw vuelve a intentar una vez los turnos vacíos compatibles con OpenAI cuya reproducción sea segura; los fallos persistentes suelen indicar que el backend emite contenido vacío o no textual, o que suprime el texto de la respuesta final.
    - Las solicitudes directas pequeñas funcionan, pero las ejecuciones del agente de OpenClaw fallan debido a bloqueos del backend o del modelo (por ejemplo, Gemma en algunas compilaciones de `inferrs`): es probable que el transporte de OpenClaw ya sea correcto; el backend falla con la estructura más grande del prompt del entorno de ejecución del agente.
    - Los fallos disminuyen al deshabilitar las herramientas, pero no desaparecen: los esquemas de herramientas contribuían a la carga, pero el problema restante sigue siendo la capacidad del modelo o servidor ascendente, o un error del backend.

  </Accordion>
  <Accordion title="Opciones de solución">
    1. Establezca `compat.requiresStringContent: true` para backends de Chat Completions que solo admitan cadenas.
    2. Establezca `compat.strictMessageKeys: true` para backends estrictos de Chat Completions que solo acepten `role` y `content` en cada mensaje.
    3. Establezca `compat.supportsTools: false` para modelos o backends que no puedan gestionar de manera fiable el conjunto de esquemas de herramientas de OpenClaw.
    4. Reduzca la carga del prompt cuando sea posible: un arranque más pequeño del espacio de trabajo, un historial de sesión más corto, un modelo local más ligero o un backend con mejor compatibilidad con contextos largos.
    5. Si las solicitudes directas pequeñas siguen funcionando, pero los turnos del agente de OpenClaw continúan bloqueándose dentro del backend, trátelo como una limitación del servidor o modelo ascendente y presente allí una reproducción con la estructura de carga útil aceptada.
  </Accordion>
</AccordionGroup>

Relacionado:

- [Configuración](/es/gateway/configuration)
- [Modelos locales](/es/gateway/local-models)
- [Endpoints compatibles con OpenAI](/es/gateway/configuration-reference#openai-compatible-endpoints)

## Sin respuestas

Si los canales están activos pero nada responde, compruebe el enrutamiento y la política antes de volver a conectar nada.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

Busque:

- Emparejamiento pendiente para remitentes de mensajes directos.
- Restricción por mención en grupos (`requireMention`, `mentionPatterns`).
- Discrepancias en la lista de permitidos del canal/grupo.

Indicadores habituales:

- `drop guild message (mention required` → el mensaje del grupo se ignora hasta que haya una mención.
- `pairing request` → el remitente necesita aprobación.
- `blocked` / `allowlist` → la política filtró al remitente/canal.

Relacionado:

- [Solución de problemas de canales](/es/channels/troubleshooting)
- [Grupos](/es/channels/groups)
- [Emparejamiento](/es/channels/pairing)

## Conectividad de la interfaz de control del panel

Cuando el panel o la interfaz de control no se conecten, valide la URL, el modo de autenticación y los supuestos del contexto seguro.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

Busque:

- URL de sondeo y URL del panel correctas.
- Discrepancia del modo de autenticación o del token entre el cliente y el gateway.
- Uso de HTTP cuando se requiere la identidad del dispositivo.

Si un navegador local no puede conectarse a `127.0.0.1:18789` después de una actualización, primero recupere el servicio Gateway local y confirme que está sirviendo el panel:

```bash
openclaw gateway restart
lsof -i :18789
curl http://127.0.0.1:18789
```

Si `curl` devuelve HTML de OpenClaw, el Gateway funciona y probablemente el problema restante sea la caché del navegador, un enlace profundo antiguo o el estado obsoleto de una pestaña. Abra `http://127.0.0.1:18789` directamente y navegue desde el panel. Si el servicio no permanece en ejecución después de reiniciarlo, ejecute `openclaw gateway start` y vuelva a comprobar `openclaw gateway status`.

<AccordionGroup>
  <Accordion title="Indicadores de conexión/autenticación">
    - `device identity required` → contexto no seguro o falta la autenticación del dispositivo.
    - `origin not allowed` → el `Origin` del navegador no está en `gateway.controlUi.allowedOrigins` (o la conexión procede de un origen de navegador que no es de bucle invertido sin una lista de permitidos explícita).
    - `device nonce required` / `device nonce mismatch` → el cliente no está completando el flujo de autenticación de dispositivo basado en desafío (`connect.challenge` + `device.nonce`).
    - `device signature invalid` / `device signature expired` → el cliente firmó la carga útil incorrecta (o una marca de tiempo obsoleta) para el protocolo de enlace actual.
    - `AUTH_TOKEN_MISMATCH` con `canRetryWithDeviceToken=true` → el cliente puede realizar un reintento de confianza con el token de dispositivo almacenado en caché.
    - Ese reintento con el token almacenado en caché reutiliza el conjunto de ámbitos almacenado con el token del dispositivo emparejado. En cambio, los llamadores con `deviceToken` explícito / `scopes` explícito conservan el conjunto de ámbitos solicitado.
    - `AUTH_SCOPE_MISMATCH` → se reconoció el token del dispositivo, pero sus ámbitos aprobados no cubren esta solicitud de conexión; vuelva a emparejar o apruebe el contrato de ámbitos solicitado en lugar de rotar un token compartido del gateway.
    - Fuera de esa ruta de reintento, la precedencia de autenticación para la conexión es: primero el token o la contraseña compartidos explícitos, después `deviceToken` explícito, luego el token de dispositivo almacenado y, por último, el token de arranque.
    - En la ruta asíncrona de Tailscale Serve para la interfaz de control, los intentos fallidos correspondientes al mismo `{scope, ip}` se serializan antes de que el limitador registre el fallo. Por tanto, dos reintentos simultáneos incorrectos del mismo cliente pueden mostrar `retry later` en el segundo intento, en vez de dos discrepancias simples.
    - `too many failed authentication attempts (retry later)` desde un cliente de bucle invertido con origen de navegador → los fallos repetidos del mismo `Origin` normalizado se bloquean temporalmente; otro origen de localhost utiliza un depósito distinto.
    - `unauthorized` repetido después de ese reintento → divergencia entre el token compartido y el token del dispositivo; actualice la configuración del token y vuelva a aprobar o rote el token del dispositivo si es necesario.
    - `gateway connect failed:` → destino de host, puerto o URL incorrecto.

  </Accordion>
</AccordionGroup>

### Mapa rápido de códigos de detalles de autenticación

Utilice `error.details.code` de la respuesta fallida de `connect` para elegir la siguiente acción:

| Código de detalle            | Significado                                                                                                                                                                                  | Acción recomendada                                                                                                                                                                                                                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | El cliente no envió un token compartido obligatorio.                                                                                                                                         | Pegue o establezca el token en el cliente y vuelva a intentarlo. Para las rutas del panel: `openclaw config get gateway.auth.token` y, después, péguelo en la configuración de la interfaz de control.                                                                                                          |
| `AUTH_TOKEN_MISMATCH`        | El token compartido no coincidió con el token de autenticación del gateway.                                                                                                                  | Si `canRetryWithDeviceToken=true`, permita un reintento de confianza. Los reintentos con tokens en caché reutilizan los ámbitos aprobados almacenados; los llamadores con `deviceToken` / `scopes` explícitos conservan los ámbitos solicitados. Si sigue fallando, ejecute la [lista de comprobación para recuperar la divergencia de tokens](/es/cli/devices#token-drift-recovery-checklist). |
| `AUTH_DEVICE_TOKEN_MISMATCH` | El token almacenado en caché de cada dispositivo está obsoleto o revocado.                                                                                                                   | Rote o vuelva a aprobar el token del dispositivo mediante la [CLI de dispositivos](/es/cli/devices) y vuelva a conectarse.                                                                                                                                                                  |
| `AUTH_SCOPE_MISMATCH`        | El token del dispositivo es válido, pero su rol o sus ámbitos aprobados no cubren esta solicitud de conexión.                                                                                | Vuelva a emparejar el dispositivo o apruebe el contrato de ámbitos solicitado; no lo trate como una divergencia del token compartido.                                                                                                                                                    |
| `PAIRING_REQUIRED`           | La identidad del dispositivo necesita aprobación. Compruebe `error.details.reason` para `not-paired`, `scope-upgrade`, `role-upgrade` o `metadata-upgrade`, y utilice `requestId` / `remediationHint` cuando estén presentes. | Apruebe la solicitud pendiente: `openclaw devices list` y, después, `openclaw devices approve <requestId>`. Las actualizaciones de ámbito o rol utilizan el mismo flujo después de revisar el acceso solicitado.                                                                                                    |

<Note>
Las RPC directas del backend de bucle invertido autenticadas con el token o la contraseña compartidos del gateway no deberían depender del conjunto básico de ámbitos del dispositivo emparejado de la CLI. Si los subagentes u otras llamadas internas siguen fallando con `scope-upgrade`, compruebe que el llamador utiliza `client.id: "gateway-client"` y `client.mode: "backend"`, y que no fuerza un `deviceIdentity` explícito ni un token de dispositivo.
</Note>

Comprobación de migración de autenticación de dispositivos v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

Si los registros muestran errores de nonce o firma, actualice el cliente que se conecta y verifíquelo:

<Steps>
  <Step title="Esperar a connect.challenge">
    El cliente espera el `connect.challenge` emitido por el gateway.
  </Step>
  <Step title="Firmar la carga útil">
    El cliente firma la carga útil vinculada al desafío.
  </Step>
  <Step title="Enviar el nonce del dispositivo">
    El cliente envía `connect.params.device.nonce` con el mismo nonce del desafío.
  </Step>
</Steps>

Si `openclaw devices rotate` / `revoke` / `remove` se deniega de forma inesperada:

- Las sesiones con token de dispositivo emparejado solo pueden gestionar **su propio** dispositivo, a menos que el llamador también tenga `operator.admin`.
- `openclaw devices rotate --scope ...` solo puede solicitar ámbitos de operador que la sesión del llamador ya posea.

Relacionado:

- [Configuración](/es/gateway/configuration) (modos de autenticación del gateway)
- [Interfaz de control](/es/web/control-ui)
- [Dispositivos](/es/cli/devices)
- [Acceso remoto](/es/gateway/remote)
- [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth)

## El servicio Gateway no se está ejecutando

Utilice esta sección cuando el servicio esté instalado, pero el proceso no permanezca activo.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # también analiza los servicios del sistema
```

Busque:

- `Runtime: stopped` con indicaciones de salida.
- Discrepancia en la configuración del servicio (`Config (cli)` frente a `Config (service)`).
- Conflictos de puerto o proceso de escucha.
- Instalaciones adicionales de launchd/systemd/schtasks cuando se utiliza `--deep`.
- Indicaciones de limpieza de `Other gateway-like services detected (best effort)`.

<AccordionGroup>
  <Accordion title="Indicadores habituales">
    - `Gateway start blocked: set gateway.mode=local` o `existing config is missing gateway.mode` → el modo de gateway local no está habilitado, o el archivo de configuración se sobrescribió y perdió `gateway.mode`. Solución: establezca `gateway.mode="local"` en la configuración o vuelva a ejecutar `openclaw onboard --mode local` / `openclaw setup` para restaurar la configuración prevista del modo local. Si ejecuta OpenClaw mediante Podman, la ruta de configuración predeterminada es `~/.openclaw/openclaw.json`.
    - `refusing to bind gateway ... without auth` → enlace que no es de bucle invertido sin una ruta válida de autenticación del gateway (token/contraseña o proxy de confianza cuando esté configurado).
    - `another gateway instance is already listening` / `EADDRINUSE` → conflicto de puerto.
    - `Other gateway-like services detected (best effort)` → existen unidades launchd/systemd/schtasks obsoletas o paralelas. La mayoría de las configuraciones deberían mantener un gateway por máquina; si necesita más de uno, aísle los puertos, la configuración, el estado y el espacio de trabajo. Consulte [/gateway#multiple-gateways-same-host](/es/gateway#multiple-gateways-same-host).
    - `System-level OpenClaw gateway service detected` de doctor → existe una unidad de sistema systemd, mientras que falta el servicio de nivel de usuario. Elimine o deshabilite el duplicado antes de permitir que doctor instale un servicio de usuario, o establezca `OPENCLAW_SERVICE_REPAIR_POLICY=external` si la unidad del sistema es el supervisor previsto.
    - `Gateway service port does not match current gateway config` → el supervisor instalado aún fija el antiguo `--port`. Ejecute `openclaw doctor --fix` o `openclaw gateway install --force` y, después, reinicie el servicio Gateway.

  </Accordion>
</AccordionGroup>

Relacionado:

- [Ejecución en segundo plano y herramienta de procesos](/es/gateway/background-process)
- [Configuración](/es/gateway/configuration)
- [Doctor](/es/gateway/doctor)

## El gateway de macOS deja de responder silenciosamente y se reanuda al interactuar con el panel

Se utiliza cuando los canales (Telegram, WhatsApp, etc.) de un host macOS dejan de responder durante periodos de entre minutos y horas, y el Gateway parece volver a funcionar en cuanto se abre la interfaz de control, se accede por SSH o se interactúa de otro modo con el host. Normalmente no hay ningún síntoma evidente en `openclaw status` porque, para cuando se revisa, el Gateway ya vuelve a estar activo.

```bash
ls ~/.openclaw/logs/stability/ | tail -5
openclaw gateway stability --bundle latest
pmset -g log | grep -iE "sleep|wake|maintenance" | tail -50
launchctl print gui/$UID/ai.openclaw.gateway | grep -E "state|last exit|runs"
```

Qué buscar:

- Uno o varios paquetes `*-uncaught_exception.json` en `~/.openclaw/logs/stability/` con `error.code` establecido en un código de red transitorio, como `ENETDOWN`, `ENETUNREACH`, `EHOSTUNREACH` o `ECONNREFUSED`.
- Líneas de `pmset -g log` como `Entering Sleep state due to 'Maintenance Sleep'` o `en0 driver is slow (msg: WillChangeState to 0)` que coincidan con las marcas de tiempo de los fallos. Power Nap / Maintenance Sleep pone brevemente el controlador Wi-Fi en el estado 0; cualquier `connect()` saliente que se produzca durante ese intervalo puede fallar con `ENETDOWN`, incluso en un host que, por lo demás, dispone de conectividad de red completa.
- Salida de `launchctl print` que muestre `state = not running` con varios `runs` recientes y un código de salida, especialmente cuando el intervalo entre el fallo y el siguiente inicio es de aproximadamente una hora en lugar de unos segundos. launchd de macOS aplica un mecanismo no documentado de protección contra reapariciones después de una ráfaga de fallos, que puede dejar de respetar `KeepAlive=true` hasta que un activador externo, como un inicio de sesión interactivo, una conexión al panel o `launchctl kickstart`, vuelva a habilitarlo.

Indicadores habituales:

- Un paquete de estabilidad cuyo `error.code` sea `ENETDOWN` o un código relacionado, con la pila de llamadas apuntando a `net` de Node, `lookupAndConnect` / `Socket.connect`. OpenClaw `2026.5.26` y las versiones posteriores clasifican estos errores como errores de red transitorios benignos, por lo que ya no se propagan al controlador superior de excepciones no capturadas; si se utiliza una versión anterior, se debe actualizar primero.
- Periodos prolongados de inactividad que terminan en cuanto se establece una conexión con la interfaz de control o se accede al host mediante SSH: la actividad visible para el usuario es lo que vuelve a habilitar el mecanismo de reaparición de launchd, no ninguna acción del panel sobre el Gateway.
- El recuento de `runs` aumenta a lo largo del día sin una línea `received SIG*; shutting down` correspondiente en `~/Library/Logs/openclaw/gateway.log`: los cierres limpios registran una señal; los fallos transitorios no.

Qué hacer:

1. **Actualizar el Gateway** si se utiliza una versión anterior a `2026.5.26`. Después de la actualización, los futuros errores `ENETDOWN` se registran como advertencias en lugar de finalizar el proceso.
2. **Reducir la actividad de suspensión de mantenimiento** en hosts Mac mini o de escritorio destinados a funcionar como servidores siempre activos:

   ```bash
   sudo pmset -a sleep 0 disksleep 0 standby 0 powernap 0
   ```

   Esto reduce considerablemente, pero no elimina por completo, la inestabilidad subyacente del controlador. El sistema todavía puede realizar algunas suspensiones de mantenimiento para conservar conexiones TCP y mantener mDNS, independientemente de estas opciones.

3. **Añadir un supervisor de actividad** para detectar rápidamente cualquier futura ráfaga de fallos que launchd deje detenida:

   ```bash
   # Ejemplo de comprobación de actividad compatible con launchd, adecuada para un cron o LaunchAgent de 5 minutos
   state=$(launchctl print gui/$UID/ai.openclaw.gateway 2>/dev/null | awk -F'= ' '/state =/ {print $2; exit}')
   if [ "$state" != "running" ]; then
     launchctl kickstart -k gui/$UID/ai.openclaw.gateway
   fi
   ```

   El objetivo es volver a habilitar externamente el mecanismo de reaparición; `KeepAlive=true` por sí solo no es suficiente en macOS después de una ráfaga de fallos.

Relacionado:

- [Notas de la plataforma macOS](/es/platforms/macos)
- [Registro](/es/logging)
- [Doctor](/es/gateway/doctor)

## Bucle del supervisor launchd de macOS con LaunchAgents duplicados de Gateway/Node

Se utiliza cuando una instalación de macOS se reinicia continuamente cada pocos segundos, las comprobaciones de estado de `openclaw`
alternan entre disponible y no disponible, y el envío a los canales se bloquea
aunque el servicio parezca estar en ejecución.

Esto se observó en instalaciones antiguas en las que tanto `ai.openclaw.gateway` como
`ai.openclaw.node` eran LaunchAgents activos y cada uno inyectaba
`OPENCLAW_LAUNCHD_LABEL`. En ese estado, OpenClaw puede detectar la
supervisión de launchd, intentar devolver el control del reinicio a launchd y caer en un bucle rápido de
`EADDRINUSE`/reaparición en lugar de mantener un único proceso de Gateway estable.

```bash
for i in 1 2 3 4; do
  ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
  sleep 10
done

openclaw gateway status --deep
openclaw node status
launchctl print gui/$UID/ai.openclaw.gateway | grep -E 'state|last exit|runs'
tail -n 80 ~/Library/Logs/openclaw/gateway.log
```

Qué buscar:

- Más de un PID del Gateway durante la muestra de 30 segundos, en lugar de un único
  proceso estable.
- `EADDRINUSE`, `another gateway instance is already listening` o líneas repetidas
  de reinicio/transferencia en `gateway.log`.
- Tanto `~/Library/LaunchAgents/ai.openclaw.gateway.plist` como
  `~/Library/LaunchAgents/ai.openclaw.node.plist` cargados al mismo tiempo en un
  host que solo debería ejecutar un servicio de Gateway administrado.

Qué hacer:

1. Si este host solo debe ejecutar el servicio Gateway, se debe eliminar el servicio
   Node administrado mediante OpenClaw. **Omitir este paso** si se depende activamente del servicio Node
   para funciones de nodos remotos; al desinstalarlo, esas funciones dejan de estar disponibles en
   este host:

   ```bash
   openclaw node uninstall
   ```

2. Instalar un contenedor persistente para el Gateway que borre los
   indicadores heredados de launchd antes de iniciar OpenClaw. Se debe utilizar la opción `--wrapper` compatible; no
   se debe editar el archivo generado en `~/.openclaw/service-env/`, ya que la reinstalación
   del servicio, las actualizaciones y las reparaciones de Doctor vuelven a generar ese archivo:

   ```bash
   mkdir -p ~/.local/bin
   cat >~/.local/bin/openclaw-launchd-workaround <<'EOF'
   #!/bin/sh
   set -eu
   unset OPENCLAW_LAUNCHD_LABEL LAUNCH_JOB_LABEL LAUNCH_JOB_NAME XPC_SERVICE_NAME || true
   exec openclaw "$@"
   EOF
   chmod 700 ~/.local/bin/openclaw-launchd-workaround

   openclaw gateway install \
     --wrapper ~/.local/bin/openclaw-launchd-workaround \
     --force
   ```

   `gateway install` conserva la ruta del contenedor entre reinstalaciones forzadas,
   actualizaciones y reparaciones de Doctor.

3. Verificar que el Gateway sea estable y preste servicio RPC, no que simplemente esté escuchando:

   ```bash
   openclaw gateway status --deep --require-rpc

   for i in 1 2 3 4; do
     ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
     sleep 10
   done
   ```

   La muestra de PID debe mostrar un único proceso estable en lugar de un conjunto rotatorio de
   PID, y el envío entrante a los canales debe reanudarse.

4. Después de actualizar a una versión en la que se haya corregido el bucle subyacente de LaunchAgents
   duplicados, se debe eliminar la solución provisional y reinstalar el servicio administrado normal:

   ```bash
   OPENCLAW_WRAPPER= openclaw gateway install --force
   rm ~/.local/bin/openclaw-launchd-workaround
   ```

Relacionado:

- [Notas de la plataforma macOS](/es/platforms/mac/bundled-gateway)
- [Doctor](/es/gateway/doctor)
- [CLI del Gateway](/es/cli/gateway)

## El Gateway se cierra durante un uso elevado de memoria

Se utiliza cuando el Gateway desaparece bajo carga, el supervisor informa de un reinicio similar a uno provocado por falta de memoria o los registros mencionan `critical memory pressure bundle written`.

```bash
openclaw gateway status --deep
openclaw logs --follow
openclaw gateway stability --bundle latest
openclaw gateway diagnostics export
```

Qué buscar:

- `Reason: diagnostic.memory.pressure.critical` en el paquete de estabilidad más reciente.
- `Memory pressure:` con `critical/rss_threshold`, `critical/heap_threshold` o `critical/rss_growth`.
- Valores de `V8 heap:` cercanos al límite del montón.
- Entradas de `Largest session files:` como `agents/<agent>/sessions/<session>.jsonl` o `sessions/<session>.jsonl`.
- Contadores de memoria de cgroup de Linux cuando el Gateway se ejecuta dentro de un contenedor o un servicio con memoria limitada.

Indicadores habituales:

- `critical memory pressure bundle written` aparece poco antes del reinicio → OpenClaw capturó un paquete de estabilidad anterior al agotamiento de memoria. Se puede inspeccionar con `openclaw gateway stability --bundle latest`.
- `memory pressure: level=critical` aparece en los registros del Gateway → OpenClaw detectó una presión crítica de memoria y registró los datos disponibles sobre la memoria del proceso.
- `Largest session files:` apunta a una ruta de transcripción censurada muy grande → se debe reducir el historial conservado de la sesión, inspeccionar su crecimiento o mover las transcripciones antiguas fuera del almacén activo antes de reiniciar.
- Los bytes utilizados de `V8 heap:` están cerca del límite del montón → primero se debe reducir la presión de las indicaciones o sesiones, o disminuir el trabajo simultáneo. En un servicio administrado, se debe inspeccionar `Gateway heap:` en `openclaw gateway status`; si indica `not set`, se deben volver a generar los metadatos antiguos del servicio con `openclaw gateway install --force`. La variable `NODE_OPTIONS` del entorno del shell se ignora intencionadamente. Solo se debe utilizar una configuración explícita del límite del montón en el supervisor después de confirmar la carga de trabajo sostenida y reservar suficiente margen para la memoria nativa.
- `Memory pressure: critical/rss_growth` → la memoria creció rápidamente dentro de un único intervalo de muestreo. Se deben revisar los registros más recientes para detectar una importación grande, una salida descontrolada de herramientas, reintentos repetidos o un lote de trabajo de agentes en cola.
- Aparece una presión crítica de memoria en los registros, pero no existe ningún paquete → se debe capturar `openclaw gateway diagnostics export` después del suceso para obtener las pruebas operativas disponibles.

El paquete de estabilidad no contiene cargas útiles. Incluye pruebas operativas sobre la memoria y rutas de archivos relativas censuradas, pero no texto de mensajes, cuerpos de Webhook, credenciales, tokens, cookies ni identificadores de sesión sin procesar. Se debe adjuntar la exportación de diagnósticos a los informes de errores en lugar de copiar los registros sin procesar.

Relacionado:

- [Estado del Gateway](/es/gateway/health)
- [Exportación de diagnósticos](/es/gateway/diagnostics)
- [Sesiones](/es/cli/sessions)

## El Gateway rechazó una configuración no válida

Se utiliza cuando el inicio del Gateway falla con `Invalid config` o cuando los registros de recarga en caliente indican que se omitió una edición no válida.

```bash
openclaw logs --follow
openclaw config file
openclaw config validate
openclaw doctor
```

Qué buscar:

- `Invalid config at ...`
- `config reload skipped (invalid config): ...`
- `Config write rejected: ...`
- Un archivo `openclaw.json.rejected.*` con marca de tiempo junto a la configuración activa.
- Un archivo `openclaw.json.clobbered.*` con marca de tiempo si `doctor --fix` reparó una edición directa defectuosa.
- OpenClaw conserva los 32 archivos `.clobbered.*` más recientes de cada ruta de configuración y rota los más antiguos.

<AccordionGroup>
  <Accordion title="Qué ocurrió">
    - La configuración no superó la validación durante el inicio, la recarga en caliente o una escritura gestionada por OpenClaw.
    - El inicio del Gateway falla de forma segura en lugar de sobrescribir `openclaw.json`.
    - La recarga en caliente omite las ediciones externas no válidas y mantiene activa la configuración actual del entorno de ejecución.
    - Las escrituras gestionadas por OpenClaw rechazan las cargas útiles no válidas o destructivas antes de confirmarlas y guardan `.rejected.*`.
    - `openclaw doctor --fix` se encarga de la reparación. Puede eliminar prefijos que no sean JSON o restaurar la última copia válida conocida, al tiempo que conserva la carga útil rechazada como `.clobbered.*`.
    - Cuando se realizan muchas reparaciones en una misma ruta de configuración, OpenClaw rota los archivos `.clobbered.*` más antiguos para que la carga útil reparada más reciente siga estando disponible.

  </Accordion>
  <Accordion title="Inspeccionar y reparar">
    ```bash
    CONFIG="$(openclaw config file)"
    ls -lt "$CONFIG".clobbered.* "$CONFIG".rejected.* 2>/dev/null | head
    diff -u "$CONFIG" "$(ls -t "$CONFIG".clobbered.* 2>/dev/null | head -n 1)"
    openclaw config validate
    openclaw doctor
    ```
  </Accordion>
  <Accordion title="Indicadores comunes">
    - `.clobbered.*` existe → doctor conservó una edición externa dañada mientras reparaba la configuración activa.
    - `.rejected.*` existe → una escritura de configuración propiedad de OpenClaw no superó las comprobaciones de esquema o sobrescritura antes de confirmarse.
    - `Config write rejected:` → la escritura intentó eliminar una estructura obligatoria, reducir drásticamente el archivo o guardar una configuración no válida.
    - `config reload skipped (invalid config):` → una edición directa no superó la validación y el Gateway en ejecución la ignoró.
    - `Invalid config at ...` → el inicio falló antes de que arrancaran los servicios del Gateway.
    - `missing-meta-vs-last-good`, `gateway-mode-missing-vs-last-good` o `size-drop-vs-last-good:*` → se rechazó una escritura propiedad de OpenClaw porque perdió campos o tamaño en comparación con la última copia de seguridad válida conocida.
    - `Config last-known-good promotion skipped` → el candidato contenía marcadores de posición de secretos censurados, como `***`.

  </Accordion>
  <Accordion title="Opciones de corrección">
    1. Ejecute `openclaw doctor --fix` para que doctor repare la configuración con prefijo o sobrescrita, o restaure la última válida conocida.
    2. Copie únicamente las claves deseadas de `.clobbered.*` o `.rejected.*` y, a continuación, aplíquelas con `openclaw config set` o `config.patch`.
    3. Ejecute `openclaw config validate` antes de reiniciar.
    4. Si edita manualmente, conserve la configuración JSON5 completa, no solo el objeto parcial que deseaba cambiar.
  </Accordion>
</AccordionGroup>

Relacionado:

- [Configuración](/es/cli/config)
- [Configuración: recarga en caliente](/es/gateway/configuration#config-hot-reload)
- [Configuración: validación estricta](/es/gateway/configuration#strict-validation)
- [Doctor](/es/gateway/doctor)

## Advertencias de la sonda del Gateway

Úselo cuando `openclaw gateway probe` llegue a algún destino, pero siga mostrando un bloque de advertencias.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

Busque:

- `warnings[].code` y `primaryTargetId` en la salida JSON.
- Si la advertencia trata sobre la alternativa mediante SSH, varios gateways, ámbitos ausentes o referencias de autenticación sin resolver.

Indicadores comunes:

- `SSH tunnel failed to start; falling back to direct probes.` → la configuración de SSH falló, pero el comando aun así intentó usar los destinos directos configurados o de bucle invertido.
- `multiple reachable gateway identities detected` → respondieron gateways distintos, o OpenClaw no pudo demostrar que los destinos accesibles fueran el mismo gateway. Un túnel SSH, una URL de proxy o una URL remota configurada hacia el mismo gateway se consideran un único gateway con varios transportes, aunque los puertos de transporte sean diferentes.
- `Read-probe diagnostics are limited by gateway scopes (missing operator.read)` → la conexión funcionó, pero el RPC de detalles está limitado por el ámbito; empareje la identidad del dispositivo o use credenciales con `operator.read`.
- `Gateway accepted the WebSocket connection, but follow-up read diagnostics failed` → la conexión funcionó, pero el conjunto completo de RPC de diagnóstico agotó el tiempo de espera o falló. Considérelo un Gateway accesible con diagnósticos degradados; compare `connect.ok` y `connect.rpcOk` en la salida de `--json`.
- `Capability: pairing-pending` o `gateway closed (1008): pairing required` → el gateway respondió, pero este cliente aún necesita emparejamiento o aprobación antes del acceso normal del operador.
- Texto de advertencia de SecretRef sin resolver para `gateway.auth.*` / `gateway.remote.*` → el material de autenticación no estaba disponible en esta ruta de comandos para el destino fallido.

Relacionado:

- [Gateway](/es/cli/gateway)
- [Varios gateways en el mismo host](/es/gateway#multiple-gateways-same-host)
- [Acceso remoto](/es/gateway/remote)

## Canal conectado, pero los mensajes no circulan

Si el estado del canal es conectado pero el flujo de mensajes está interrumpido, céntrese en la política, los permisos y las reglas de entrega específicas del canal.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

Busque:

- Política de mensajes directos (`pairing`, `allowlist`, `open`, `disabled`).
- Lista de permitidos de grupos y requisitos de mención.
- Permisos o ámbitos de API del canal ausentes.

Indicadores comunes:

- `mention required` → la política de menciones del grupo ignoró el mensaje.
- `pairing` / rastros de aprobación pendiente → el remitente no está aprobado.
- `missing_scope`, `not_in_channel`, `Forbidden`, `401/403` → problema de autenticación o permisos del canal.

Relacionado:

- [Solución de problemas de canales](/es/channels/troubleshooting)
- [Discord](/es/channels/discord)
- [Telegram](/es/channels/telegram)
- [WhatsApp](/es/channels/whatsapp)

## Entrega de Cron y Heartbeat

Si Cron o Heartbeat no se ejecutaron o no realizaron la entrega, verifique primero el estado del planificador y después el destino de entrega.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

Busque:

- Cron habilitado y la siguiente activación presente.
- Estado del historial de ejecución del trabajo (`ok`, `skipped`, `error`).
- Motivos por los que se omitió Heartbeat (`quiet-hours`, `requests-in-flight`, `cron-in-progress`, `lanes-busy`, `alerts-disabled`, `empty-heartbeat-file`).

<AccordionGroup>
  <Accordion title="Indicadores comunes">
    - `cron: scheduler disabled; jobs will not run automatically` → Cron deshabilitado.
    - `cron: timer tick failed` → el ciclo del planificador falló; compruebe si hay errores de archivos, registros o tiempo de ejecución.
    - `heartbeat skipped` con `reason=quiet-hours` → fuera del intervalo de horas activas.
    - `heartbeat skipped` con `reason=empty-heartbeat-file` → el borrador del monitor de Heartbeat solo contiene espacios en blanco, comentarios, encabezados, delimitadores de bloque o una estructura de lista de comprobación vacía, por lo que OpenClaw omite la llamada al modelo.
    - `heartbeat: unknown accountId` → id. de cuenta no válido para el destino de entrega de Heartbeat.
    - `heartbeat skipped` con `reason=dm-blocked` → el destino de Heartbeat se resolvió como un destino de tipo mensaje directo mientras `agents.defaults.heartbeat.directPolicy` (o la anulación por agente) está establecido en `block`.

  </Accordion>
</AccordionGroup>

Relacionado:

- [Heartbeat](/es/gateway/heartbeat)
- [Tareas programadas](/es/automation/cron-jobs)
- [Tareas programadas: solución de problemas](/es/automation/cron-jobs#troubleshooting)

## Node emparejado, pero la herramienta falla

Si un Node está emparejado pero las herramientas fallan, aísle el estado de primer plano, permisos y aprobaciones.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

Busque:

- Node en línea con las capacidades esperadas.
- Concesiones de permisos del sistema operativo para cámara, micrófono, ubicación y pantalla.
- Aprobaciones de ejecución y estado de la lista de permitidos.

Indicadores comunes:

- `NODE_BACKGROUND_UNAVAILABLE` → la aplicación del Node debe estar en primer plano.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` → falta un permiso del sistema operativo.
- `SYSTEM_RUN_DENIED: approval required` → aprobación de ejecución pendiente.
- `SYSTEM_RUN_DENIED: allowlist miss` → comando bloqueado por la lista de permitidos.

Relacionado:

- [Aprobaciones de ejecución](/es/tools/exec-approvals)
- [Solución de problemas de Node](/es/nodes/troubleshooting)
- [Nodes](/es/nodes/index)

## La herramienta de navegador falla

Úselo cuando las acciones de la herramienta de navegador fallen aunque el gateway esté en buen estado.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

Busque:

- Si `plugins.allow` está establecido e incluye `browser`.
- Ruta válida al ejecutable del navegador.
- Accesibilidad del perfil CDP.
- Disponibilidad local de Chrome para los perfiles `existing-session` / `user`.

<AccordionGroup>
  <Accordion title="Indicadores del Plugin o ejecutable">
    - `unknown command "browser"` o `unknown command 'browser'` → `plugins.allow` excluye el Plugin de navegador incluido.
    - Herramienta de navegador ausente o no disponible mientras `browser.enabled=true` → `plugins.allow` excluye `browser`, por lo que el Plugin nunca se cargó.
    - `Failed to start Chrome CDP on port` → el proceso del navegador no pudo iniciarse.
    - `browser.executablePath not found` → la ruta configurada no es válida.
    - `browser.cdpUrl must be http(s) or ws(s)` → la URL de CDP configurada utiliza un esquema no compatible, como `file:` o `ftp:`.
    - `browser.cdpUrl has invalid port` → la URL de CDP configurada tiene un puerto no válido o fuera de rango.
    - `Playwright is not available in this gateway build; '<feature>' is unsupported.` → la instalación actual del Gateway carece de la dependencia principal del entorno de ejecución del navegador; reinstale o actualice OpenClaw y, a continuación, reinicie el Gateway. Las instantáneas ARIA y las capturas de pantalla básicas de páginas aún pueden funcionar, pero la navegación, las instantáneas de IA, las capturas de pantalla de elementos mediante selectores CSS y la exportación a PDF permanecen no disponibles.

  </Accordion>
  <Accordion title="Indicadores de Chrome MCP o sesiones existentes">
    - `Could not find DevToolsActivePort for chrome` → la sesión existente de Chrome MCP aún no pudo conectarse al directorio de datos del navegador seleccionado. Abra la página de inspección del navegador, habilite la depuración remota, mantenga abierto el navegador, apruebe la primera solicitud de conexión y vuelva a intentarlo. Si no se requiere el estado de sesión iniciada, es preferible el perfil administrado `openclaw`.
    - `No browser tabs found for profile="user"` → el perfil de conexión de Chrome MCP no tiene pestañas locales de Chrome abiertas.
    - `Remote CDP for profile "<name>" is not reachable` → el host del Gateway no puede acceder al punto de conexión CDP remoto configurado.
    - `Browser attachOnly is enabled ... not reachable` o `Browser attachOnly is enabled and CDP websocket ... is not reachable` → el perfil exclusivo para conexiones no tiene ningún destino accesible, o el punto de conexión HTTP respondió pero aun así no se pudo abrir el WebSocket de CDP.

  </Accordion>
  <Accordion title="Indicadores de elementos, capturas de pantalla o cargas">
    - `fullPage is not supported for element screenshots` → la solicitud de captura de pantalla combinó `--full-page` con `--ref` o `--element`.
    - `element screenshots are not supported for existing-session profiles; use ref from snapshot.` → las llamadas de captura de pantalla de Chrome MCP / `existing-session` deben usar la captura de página o una `--ref` de instantánea, no un `--element` CSS.
    - `existing-session file uploads do not support element selectors; use ref/inputRef.` → los enlaces de carga de Chrome MCP necesitan referencias de instantáneas, no selectores CSS.
    - `existing-session file uploads currently support one file at a time.` → envíe una carga por llamada en los perfiles de Chrome MCP.
    - `existing-session dialog handling does not support timeoutMs.` → los enlaces de diálogo de los perfiles de Chrome MCP no admiten anulaciones del tiempo de espera.
    - `existing-session type does not support timeoutMs overrides.` → omita `timeoutMs` para `act:type` en los perfiles `profile="user"` / de sesión existente de Chrome MCP, o use un perfil de navegador administrado o CDP cuando se requiera un tiempo de espera personalizado.
    - `response body is not supported for existing-session profiles yet.` → `responsebody` aún requiere un navegador administrado o un perfil CDP sin procesar.
    - Anulaciones obsoletas de área de visualización, modo oscuro, configuración regional o modo sin conexión en perfiles exclusivos para conexiones o CDP remotos → ejecute `openclaw browser stop --browser-profile <name>` para cerrar la sesión de control activa y liberar el estado de emulación de Playwright/CDP sin reiniciar todo el Gateway.

  </Accordion>
</AccordionGroup>

Relacionado:

- [Navegador (administrado por OpenClaw)](/es/tools/browser)
- [Solución de problemas del navegador](/es/tools/browser-linux-troubleshooting)

## Si actualizó y algo dejó de funcionar de repente

La mayoría de los fallos posteriores a una actualización se deben a desviaciones en la configuración o a que ahora se aplican valores predeterminados más estrictos.

<AccordionGroup>
  <Accordion title="1. Cambió el comportamiento de las anulaciones de autenticación y URL">
    ```bash
    openclaw gateway status
    openclaw config get gateway.mode
    openclaw config get gateway.remote.url
    openclaw config get gateway.auth.mode
    ```

    Qué comprobar:

    - Si `gateway.mode=remote`, las llamadas de la CLI pueden estar apuntando al servicio remoto aunque el servicio local funcione correctamente.
    - Las llamadas explícitas a `--url` no recurren a las credenciales almacenadas.

    Indicadores habituales:

    - `gateway connect failed:` → destino de URL incorrecto.
    - `unauthorized` → el endpoint es accesible, pero la autenticación es incorrecta.

  </Accordion>
  <Accordion title="2. Las medidas de seguridad de vinculación y autenticación son más estrictas">
    ```bash
    openclaw config get gateway.bind
    openclaw config get gateway.auth.mode
    openclaw config get gateway.auth.token
    openclaw gateway status
    openclaw logs --follow
    ```

    Qué comprobar:

    - Las vinculaciones que no sean de bucle invertido (`lan`, `tailnet`, `custom`) necesitan una vía válida de autenticación del Gateway: autenticación mediante token compartido o contraseña, o una implementación `trusted-proxy` que no sea de bucle invertido y esté configurada correctamente.
    - Las claves antiguas como `gateway.token` no sustituyen a `gateway.auth.token`.

    Indicadores habituales:

    - `refusing to bind gateway ... without auth` → vinculación que no es de bucle invertido sin una vía válida de autenticación del Gateway.
    - `Connectivity probe: failed` mientras el entorno de ejecución está en funcionamiento → el Gateway está activo, pero no se puede acceder a él con la autenticación o la URL actuales.

  </Accordion>
  <Accordion title="3. El estado del emparejamiento y de la identidad del dispositivo ha cambiado">
    ```bash
    openclaw devices list
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    openclaw doctor
    ```

    Qué comprobar:

    - Aprobaciones de dispositivos pendientes para el panel de control o los nodos.
    - Aprobaciones de emparejamiento por mensaje directo pendientes tras cambios en las políticas o la identidad.

    Indicadores habituales:

    - `device identity required` → no se ha satisfecho la autenticación del dispositivo.
    - `pairing required` → se debe aprobar el remitente o el dispositivo.

  </Accordion>
</AccordionGroup>

Si la configuración del servicio y el entorno de ejecución siguen sin coincidir después de las comprobaciones, reinstale los metadatos del servicio desde el mismo perfil o directorio de estado:

```bash
openclaw gateway install --force
openclaw gateway restart
```

Contenido relacionado:

- [Autenticación](/es/gateway/authentication)
- [Ejecución en segundo plano y herramienta de procesos](/es/gateway/background-process)
- [Emparejamiento de nodos](/es/gateway/pairing)

## Contenido relacionado

- [Doctor](/es/gateway/doctor)
- [Preguntas frecuentes](/es/help/faq)
- [Guía operativa del Gateway](/es/gateway)
