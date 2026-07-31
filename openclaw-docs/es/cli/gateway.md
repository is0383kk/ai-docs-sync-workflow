---
read_when:
    - Ejecución del Gateway desde la CLI (desarrollo o servidores)
    - Depuración de la autenticación, los modos de enlace y la conectividad del Gateway
    - Descubrimiento de gateways mediante Bonjour (DNS-SD local y de área amplia)
    - Integración de un supervisor externo de procesos del Gateway
sidebarTitle: Gateway
summary: CLI del Gateway de OpenClaw (`openclaw gateway`) — ejecutar, consultar y descubrir gateways
title: Gateway
x-i18n:
    generated_at: "2026-07-26T05:03:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

El Gateway es el servidor WebSocket de OpenClaw (canales, nodos, sesiones, hooks). Todos los subcomandos que aparecen a continuación se encuentran bajo `openclaw gateway ...`.

<CardGroup cols={3}>
  <Card title="Detección Bonjour" href="/es/gateway/bonjour">
    Configuración de mDNS local y DNS-SD de área amplia.
  </Card>
  <Card title="Descripción general de la detección" href="/es/gateway/discovery">
    Cómo OpenClaw anuncia y encuentra gateways.
  </Card>
  <Card title="Configuración" href="/es/gateway/configuration">
    Claves de configuración de nivel superior del Gateway.
  </Card>
</CardGroup>

## Ejecutar el Gateway

```bash
openclaw gateway
openclaw gateway run   # forma equivalente y explícita
```

<AccordionGroup>
  <Accordion title="Comportamiento de inicio">
    - Se niega a iniciarse a menos que `gateway.mode=local` esté establecido en `~/.openclaw/openclaw.json`. Use `--allow-unconfigured` para ejecuciones ad hoc o de desarrollo; omite la protección sin escribir ni reparar la configuración.
    - Cuando al iniciarse encuentra una configuración no válida que se puede reparar, un terminal interactivo ofrece ejecutar `openclaw doctor --fix` y, tras obtener el consentimiento, vuelve a intentar el inicio una vez. Las ejecuciones no interactivas nunca realizan reparaciones automáticamente; en su lugar, muestran el comando. Si la configuración reparada sigue sin ser válida, el inicio permanece detenido.
    - `openclaw onboard --mode local` y `openclaw setup` escriben `gateway.mode=local`. Si el archivo de configuración existe pero falta `gateway.mode`, se considera que la configuración está dañada o sobrescrita, y el Gateway se niega a deducir `local` — vuelva a ejecutar la incorporación, establezca la clave manualmente o pase `--allow-unconfigured`.
    - Se bloquea la vinculación más allá de la interfaz de bucle invertido sin autenticación.
    - Actualmente, los valores `lan`, `tailnet` y `custom` de `--bind` se resuelven mediante rutas que solo usan IPv4; las configuraciones con host propio que solo admiten IPv6 necesitan un proceso auxiliar IPv4 o un proxy delante del Gateway.
    - `SIGUSR1` activa un reinicio dentro del proceso cuando está autorizado. `commands.restart` (valor predeterminado: habilitado) controla los `SIGUSR1` enviados externamente; establézcalo en `false` para bloquear los reinicios manuales mediante señales del sistema operativo. La herramienta `gateway` orientada a agentes es de solo lectura; los agentes solicitan el reinicio mediante la herramienta de delegación `openclaw` aprobada por una persona.
    - `SIGINT`/`SIGTERM` detienen el proceso, pero no restauran el estado personalizado del terminal; si encapsula la CLI en una TUI o una entrada en modo sin procesar, restaure el terminal antes de salir.

  </Accordion>
</AccordionGroup>

### Opciones

<ParamField path="--port <port>" type="number">
  Puerto WebSocket (valor predeterminado procedente de la configuración o del entorno; normalmente `18789`).
</ParamField>
<ParamField path="--bind <mode>" type="string">
  Modo de vinculación: `loopback` (predeterminado), `lan`, `tailnet`, `auto`, `custom`.
</ParamField>
<ParamField path="--token <token>" type="string">
  Token compartido para `connect.params.auth.token`. El valor predeterminado es `OPENCLAW_GATEWAY_TOKEN` cuando está establecido.
</ParamField>
<ParamField path="--auth <mode>" type="string">
  Modo de autenticación: `none`, `token`, `password`, `trusted-proxy`.
</ParamField>
<ParamField path="--password <password>" type="string">
  Contraseña para `--auth password`.
</ParamField>
<ParamField path="--password-file <path>" type="string">
  Leer la contraseña del Gateway desde un archivo.
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  Exposición mediante Tailscale: `off`, `serve`, `funnel`.
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  Restablecer la configuración de serve/funnel de Tailscale al apagarse.
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  Iniciar sin exigir `gateway.mode=local`. Solo para el arranque ad hoc o de desarrollo; no conserva ni repara la configuración.
</ParamField>
<ParamField path="--dev" type="boolean">
  Crear una configuración y un espacio de trabajo de desarrollo si no existen (omite `BOOTSTRAP.md`).
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  Permitir que un Gateway de desarrollo configure automáticamente los canales a partir de variables de entorno disponibles. Requiere `--dev`.
</ParamField>
<ParamField path="--reset" type="boolean">
  Restablecer la configuración de desarrollo, las credenciales, las sesiones y el espacio de trabajo. Requiere `--dev`.
</ParamField>
<ParamField path="--force" type="boolean">
  Finalizar cualquier proceso que esté escuchando en el puerto de destino antes de iniciar. En un shell no interactivo, esta opción se niega a finalizar un proceso de escucha verificado del Gateway; use `--dev` o un `--profile` aislado con un puerto libre.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Registro detallado en stdout/stderr.
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  Mostrar únicamente los registros del backend de la CLI en la consola (también habilita stdout/stderr).
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  Estilo de registro de WebSocket: `auto`, `full`, `compact`.
</ParamField>
<ParamField path="--compact" type="boolean">
  Alias de `--ws-log compact`.
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  Registrar en JSONL los eventos sin procesar del flujo del modelo.
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  Ruta del flujo sin procesar en JSONL.
</ParamField>

`--claude-cli-logs` es un alias obsoleto de `--cli-backend-logs`.

Para `--bind custom`, establezca `gateway.customBindHost` en una dirección IPv4. Cualquier dirección distinta de `127.0.0.1` o `0.0.0.0` también requiere `127.0.0.1` en el mismo puerto para los clientes del mismo host; el inicio falla si alguno de los procesos de escucha no puede vincularse. El comodín `0.0.0.0` no añade un alias obligatorio independiente. Las configuraciones con host propio que solo admiten IPv6 necesitan un proceso auxiliar IPv4 o un proxy delante del Gateway.

## Reiniciar el Gateway

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe` solicita al Gateway en ejecución que compruebe previamente el trabajo activo y programe un único reinicio consolidado después de que termine ese trabajo. La espera está limitada a 5 minutos; cuando se agota el tiempo asignado, se fuerza el reinicio. `--safe` no se puede combinar con `--force` ni `--wait`.

`--skip-deferral` omite la protección de aplazamiento por trabajo activo durante un reinicio seguro, por lo que el Gateway se reinicia inmediatamente incluso si se notifican bloqueos. Requiere `--safe`; úselo cuando un aplazamiento quede atascado debido a una tarea descontrolada.

`--wait <duration>` sustituye el tiempo asignado al vaciado para un reinicio normal (no seguro). Acepta milisegundos sin unidad o los sufijos de unidad `ms`, `s`, `m`, `h`, `d` (por ejemplo, `30s`, `5m`, `1h30m`); `--wait 0` espera indefinidamente. No es compatible con `--force` ni `--safe`.

`--force` omite el vaciado del trabajo activo y reinicia inmediatamente. `restart` sin opciones mantiene el comportamiento de reinicio existente del gestor de servicios.

<Warning>
El uso de `--password` en línea puede quedar expuesto en los listados de procesos locales. Se recomienda usar `--password-file`, el entorno o un `gateway.auth.password` respaldado por SecretRef.
</Warning>

### Supervisores externos

Establezca `OPENCLAW_SUPERVISOR_MODE=external` únicamente cuando otro gestor de procesos controle el ciclo de vida del Gateway. En este modo:

- `openclaw gateway restart` conserva el comportamiento existente de espera segura, forzada y limitada, pero actúa sobre el Gateway en ejecución verificado en lugar de launchd, systemd o el Programador de tareas.
- Las operaciones nativas de instalación, inicio, detención y desinstalación del servicio se rechazan y se indica que se debe usar el supervisor externo.
- La actualización automática de OpenClaw se rechaza para que el supervisor pueda detener el Gateway, sustituir y finalizar el entorno de ejecución y reiniciarlo de forma segura.
- Un reinicio en un proceso nuevo escribe una transferencia limitada en SQLite antes de una salida limpia. Si falla la persistencia, el Gateway recurre a un reinicio dentro del proceso en lugar de salir sin una transferencia utilizable.

`OPENCLAW_SERVICE_REPAIR_POLICY=external` sigue siendo una política de reparación independiente de Doctor. No declara la propiedad del entorno de ejecución; los supervisores que necesiten ambos comportamientos deben establecer ambas variables.

Los supervisores externos pueden negociar y consumir transferencias de reinicio mediante el contrato interno para máquinas:

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

La versión de protocolo `1` admite la operación `consume`. El consumo valida el PID esperado y los campos limitados de la transferencia dentro de una única transacción inmediata de SQLite. Una transferencia aceptada se elimina antes de devolver el resultado satisfactorio, por lo que dos consumidores simultáneos o repetidos no pueden aceptarla. Una discrepancia del PID se conserva para el propietario correspondiente; las filas ausentes, caducadas o no válidas no autorizan un reinicio.

Las solicitudes válidas para máquinas devuelven JSON con el código de salida `0`, incluidos los resultados que no provocan un reinicio. Los argumentos no válidos devuelven `reason: "invalid-expected-pid"` con el código de salida `2`; los errores del almacén de estado devuelven `reason: "store-unavailable"` con el código de salida `1`. Los supervisores deben consultar `capabilities` en el entorno de ejecución o iniciador exacto que vayan a usar, en lugar de deducir la compatibilidad a partir de una cadena de versión de OpenClaw o leer directamente el esquema privado de SQLite.

### Perfilado del Gateway

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1` registra los tiempos de las fases durante el inicio, incluidos el retraso `eventLoopMax` por fase y los tiempos de las tablas de consulta de plugins (índice de instalaciones, registro de manifiestos, planificación del inicio y trabajo del mapa de propietarios).
- `OPENCLAW_GATEWAY_RESTART_TRACE=1` registra líneas `restart trace:` correspondientes al reinicio: gestión de señales, vaciado del trabajo activo, fases de apagado, siguiente inicio, tiempo hasta estar listo y métricas de memoria.
- `OPENCLAW_DIAGNOSTICS=timeline` con `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` escribe, con el máximo esfuerzo posible, una cronología JSONL de diagnósticos de inicio para sistemas externos de QA (equivale a la configuración `diagnostics.flags: ["timeline"]`; la ruta sigue estando disponible únicamente mediante el entorno). Añada `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` para incluir muestras del bucle de eventos.
- `pnpm build` y, a continuación, `pnpm test:startup:gateway -- --runs 5 --warmup 1` realizan una prueba de rendimiento del inicio del Gateway comparándolo con el punto de entrada compilado de la CLI: primera salida del proceso, `/healthz`, `/readyz`, tiempos del seguimiento de inicio, retraso del bucle de eventos y tiempo de la tabla de consulta de plugins.
- `pnpm build` y, a continuación, `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5` realizan una prueba de rendimiento del reinicio dentro del proceso en macOS o Linux (no se admite en Windows; el reinicio requiere `SIGUSR1`). Usa `SIGUSR1`, habilita ambos seguimientos en el proceso secundario y registra el siguiente `/healthz`, el siguiente `/readyz`, el tiempo de inactividad, el tiempo hasta estar listo, la CPU, la RSS y las métricas de seguimiento del reinicio.
- `/healthz` indica actividad; `/readyz` indica que está listo para usarse. Trate las líneas de seguimiento y la salida de las pruebas de rendimiento como una señal para atribuir la responsabilidad, no como una conclusión completa sobre el rendimiento basada en un único intervalo o una única muestra.

## Consultar un Gateway en ejecución

Todos los comandos de consulta usan RPC mediante WebSocket.

<Tabs>
  <Tab title="Modos de salida">
    - Predeterminado: legible para personas (con colores en una TTY).
    - `--json`: JSON legible por máquinas (sin estilos ni indicador de progreso).
    - `--no-color` (o `NO_COLOR=1`): deshabilita ANSI, pero conserva el diseño legible para personas.

  </Tab>
  <Tab title="Opciones compartidas">
    - `--url <url>`: URL WebSocket del Gateway.
    - `--token <token>`: token del Gateway.
    - `--password <password>`: contraseña del Gateway.
    - `--timeout <ms>`: tiempo de espera o límite (el valor predeterminado varía según el comando; consulte cada comando a continuación).
    - `--expect-final`: esperar una respuesta «final» (llamadas de agentes).

  </Tab>
</Tabs>

<Note>
Cuando se establece `--url`, la CLI no recurre a las credenciales de la configuración ni del entorno. Pase explícitamente `--token` o `--password`. La ausencia de credenciales explícitas es un error.
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` es una sonda de actividad: responde en cuanto el servidor puede atender solicitudes HTTP. `/readyz` es más estricta y permanece en rojo mientras los procesos auxiliares de plugins, los canales o los hooks configurados durante el inicio aún se están estabilizando. Las respuestas detalladas locales o autenticadas de `/readyz` incluyen un bloque de diagnóstico `eventLoop` (retraso, utilización, proporción de núcleos de CPU, indicador `degraded`).

<ParamField path="--port <port>" type="number">
  Apunta a un Gateway de bucle invertido local en este puerto. Anula `OPENCLAW_GATEWAY_URL` y `OPENCLAW_GATEWAY_PORT` para esta llamada.
</ParamField>

### `gateway usage-cost`

Obtiene resúmenes de costes de uso a partir de los registros de sesión.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  Número de días que se incluirán.
</ParamField>
<ParamField path="--agent <id>" type="string">
  Limita el resumen a un id de agente configurado.
</ParamField>
<ParamField path="--all-agents" type="boolean">
  Agrega todos los agentes configurados. No se puede combinar con `--agent`.
</ParamField>

### `gateway stability`

Obtiene el registro reciente de estabilidad de diagnóstico de un Gateway en ejecución.

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  Número máximo de eventos recientes que se incluirán (máx. `1000`).
</ParamField>
<ParamField path="--type <type>" type="string">
  Filtra por tipo de evento de diagnóstico, por ejemplo, `payload.large` o `diagnostic.memory.pressure`.
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  Incluye únicamente los eventos posteriores a un número de secuencia de diagnóstico.
</ParamField>
<ParamField path="--bundle [path]" type="string">
  Lee un paquete de estabilidad persistido en lugar de llamar al Gateway en ejecución. `--bundle latest` (o simplemente `--bundle`) selecciona el paquete más reciente del directorio de estado; también se puede proporcionar directamente la ruta de un paquete JSON.
</ParamField>
<ParamField path="--export" type="boolean">
  Escribe un archivo zip compartible con diagnósticos de soporte en lugar de mostrar los detalles de estabilidad.
</ParamField>
<ParamField path="--output <path>" type="string">
  Ruta de salida para `--export`.
</ParamField>

<AccordionGroup>
  <Accordion title="Privacidad y comportamiento de los paquetes">
    - Los registros conservan metadatos operativos: nombres de eventos, recuentos, tamaños en bytes, lecturas de memoria, estado de colas y sesiones, ids de aprobación, nombres de canales y plugins, y resúmenes de sesión censurados. Excluyen texto de chat, cuerpos de webhooks, salidas de herramientas, cuerpos de solicitudes y respuestas sin procesar, tokens, cookies, valores secretos, nombres de host e ids de sesión sin procesar. Establezca `diagnostics.enabled: false` para desactivar por completo el registro.
    - Las salidas fatales del Gateway, los tiempos de espera de apagado y los fallos de inicio tras un reinicio escriben la misma instantánea de diagnóstico en `~/.openclaw/logs/stability/openclaw-stability-*.json` cuando el registro contiene eventos. Inspeccione el paquete más reciente con `openclaw gateway stability --bundle latest`; `--limit`, `--type` y `--since-seq` también se aplican a la salida de los paquetes.

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

Escribe un archivo zip local de diagnósticos diseñado para informes de errores. Para consultar el modelo de privacidad y el contenido de los paquetes, véase [Exportación de diagnósticos](/es/gateway/diagnostics).

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  Ruta del archivo zip de salida. De forma predeterminada, se usa una exportación de soporte en el directorio de estado.
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  Número máximo de líneas de registro saneadas que se incluirán.
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  Número máximo de bytes de registro que se inspeccionarán.
</ParamField>
<ParamField path="--url <url>" type="string">
  URL de WebSocket del Gateway para la instantánea de estado.
</ParamField>
<ParamField path="--token <token>" type="string">
  Token del Gateway para la instantánea de estado.
</ParamField>
<ParamField path="--password <password>" type="string">
  Contraseña del Gateway para la instantánea de estado.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  Tiempo de espera de la instantánea de estado y salud.
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  Omite la búsqueda de paquetes de estabilidad persistidos.
</ParamField>
<ParamField path="--json" type="boolean">
  Muestra como JSON la ruta escrita, el tamaño y el manifiesto.
</ParamField>

La exportación agrupa: `manifest.json` (inventario de archivos), `summary.md` (resumen en Markdown), `diagnostics.json` (resumen de nivel superior de configuración, registros, detección, estabilidad, estado y salud), `config/sanitized.json`, `status/gateway-status.json`, `health/gateway-health.json`, `logs/openclaw-sanitized.jsonl` y `stability/latest.json` cuando existe un paquete.

Está diseñada para compartirse. Conserva detalles operativos útiles para la depuración —campos de registro seguros, nombres de subsistemas, códigos de estado, duraciones, modos configurados, puertos, ids de plugins y proveedores, ajustes de funciones no secretos y mensajes operativos de registro censurados— y omite o censura texto de chat, cuerpos de webhooks, salidas de herramientas, credenciales, cookies, identificadores de cuentas y mensajes, texto de instrucciones y prompts, nombres de host y valores secretos. Cuando un mensaje de registro parece contener texto de carga útil de usuario, chat o herramienta (por ejemplo, "el usuario dijo", "texto del chat", "salida de la herramienta" o "cuerpo del webhook"), la exportación conserva únicamente el hecho de que se omitió un mensaje y su recuento de bytes.

### `gateway status`

Muestra el servicio del Gateway (launchd/systemd/schtasks), además de una sonda opcional de conectividad y autenticación.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  Añade un destino de sonda explícito. También se siguen sondeando el destino remoto configurado y localhost.
</ParamField>
<ParamField path="--token <token>" type="string">
  Autenticación mediante token para la sonda.
</ParamField>
<ParamField path="--password <password>" type="string">
  Autenticación mediante contraseña para la sonda.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Tiempo de espera de la sonda.
</ParamField>
<ParamField path="--no-probe" type="boolean">
  Omite la sonda de conectividad (vista exclusiva del servicio).
</ParamField>
<ParamField path="--deep" type="boolean">
  Examina también los servicios del sistema.
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  Amplía la sonda de conectividad a una sonda de lectura y finaliza con un código distinto de cero si falla. No se puede combinar con `--no-probe`.
</ParamField>

<AccordionGroup>
  <Accordion title="Semántica del estado">
    - Permanece disponible para realizar diagnósticos aunque falte la configuración local de la CLI o esta no sea válida.
    - La salida predeterminada demuestra el estado del servicio, la conexión WebSocket y la capacidad de autenticación visible durante el protocolo de enlace, pero no las operaciones de lectura, escritura o administración.
    - Las sondas no realizan cambios para la autenticación inicial de dispositivos: reutilizan un token de dispositivo almacenado en caché cuando existe, pero nunca crean una nueva identidad de dispositivo de la CLI ni un registro de emparejamiento de solo lectura únicamente para comprobar el estado.
    - Resuelve las SecretRefs de autenticación configuradas para autenticar la sonda cuando es posible. Si una SecretRef obligatoria no se puede resolver, `--json` informa de `rpc.authWarning` cuando falla la conectividad o autenticación de la sonda; proporcione `--token`/`--password` explícitamente o corrija el origen del secreto. Las advertencias sobre autenticación sin resolver se suprimen cuando la sonda funciona correctamente.
    - La salida JSON incluye `gateway.version` cuando el Gateway en ejecución lo proporciona; `--require-rpc` puede recurrir a la carga útil RPC de `status.runtimeVersion` si la sonda del protocolo de enlace no puede proporcionar metadatos de versión.
    - Utilice `--require-rpc` en scripts y automatizaciones cuando no baste con que el servicio esté escuchando y también sea necesario que RPC con ámbito de lectura funcione correctamente.
    - `--deep` busca instalaciones adicionales de launchd/systemd/schtasks; cuando se encuentran varios servicios similares a un gateway, la salida para personas muestra sugerencias de limpieza (normalmente, ejecutar un gateway por máquina) e informa de una transferencia de reinicio reciente del supervisor cuando corresponde.
    - `--deep` también ejecuta la validación de la configuración en modo compatible con plugins (`pluginValidation: "full"`) y muestra advertencias del manifiesto de plugins (por ejemplo, la ausencia de metadatos de configuración del canal). El valor predeterminado `gateway status` conserva la ruta rápida de solo lectura que omite la validación de plugins.
    - La salida para personas incluye la ruta resuelta del archivo de registro, además de las rutas y la validez de la configuración de la CLI y del servicio, para ayudar a diagnosticar divergencias del perfil o del directorio de estado.
    - La salida para personas incluye `Gateway heap:` con el límite aplicado y su cálculo adaptativo. La salida JSON presenta el mismo informe como `service.gatewayHeap`.

  </Accordion>
  <Accordion title="Comprobaciones de divergencia de autenticación de systemd en Linux">
    - Las comprobaciones de divergencia de autenticación del servicio leen tanto `Environment=` como `EnvironmentFile=` de la unidad (incluidos `%h`, rutas entre comillas, varios archivos y archivos `-` opcionales).
    - Resuelve las SecretRefs de `gateway.auth.token` mediante el entorno de ejecución combinado (primero el entorno del comando del servicio y después el entorno del proceso como alternativa).
    - Las comprobaciones de divergencia de tokens omiten la resolución del token de configuración cuando la autenticación mediante token no está activa de forma efectiva (`gateway.auth.mode` establecido explícitamente en `password`/`none`/`trusted-proxy`, o modo sin establecer cuando la contraseña puede prevalecer y ningún token candidato puede hacerlo).

  </Accordion>
</AccordionGroup>

### `gateway probe`

El comando para «depurarlo todo». Siempre sondea:

- el gateway remoto configurado (si se ha establecido), y
- localhost (bucle invertido), **aunque haya un destino remoto configurado**.

Al proporcionar `--url`, ese destino explícito se añade antes de ambos. La salida para personas etiqueta los destinos como `URL (explicit)`, `Remote (configured)` / `Remote (configured, inactive)` y `Local loopback`.

<Note>
Si se puede acceder a varios destinos de sonda, se muestran todos. Un túnel SSH, una URL de TLS/proxy y una URL remota configurada pueden apuntar al mismo gateway, incluso con distintos puertos de transporte; `multiple_gateways` se reserva para gateways accesibles distintos o cuya identidad sea ambigua. Se admite la ejecución de varios gateways para perfiles aislados (por ejemplo, un bot de rescate), pero la mayoría de las instalaciones ejecutan un único gateway.
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  Utiliza este puerto para el destino de la sonda de bucle invertido local y el puerto remoto del túnel SSH. Sin `--url`, selecciona únicamente el destino de bucle invertido local en lugar de la URL del entorno del gateway configurado, el puerto del entorno o los destinos remotos.
</ParamField>

<AccordionGroup>
  <Accordion title="Interpretación">
    - `Reachable: yes` significa que al menos un destino aceptó una conexión WebSocket.
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` informa de lo que la sonda pudo demostrar sobre la autenticación, por separado de la accesibilidad.
    - `Read probe: ok` significa que las llamadas RPC detalladas con ámbito de lectura (`health`/`status`/`system-presence`/`config.get`) también se completaron correctamente.
    - `Read probe: limited - missing scope: operator.read` significa que la conexión se realizó correctamente, pero RPC con ámbito de lectura está limitado. Se informa como accesibilidad **degradada**, no como fallo total.
    - `Read probe: failed` después de `Connect: ok` significa que se estableció la conexión WebSocket, pero los diagnósticos de lectura posteriores agotaron el tiempo de espera o fallaron; también se considera un estado **degradado**, no inaccesible.
    - Al igual que `gateway status`, la sonda reutiliza la autenticación de dispositivo existente en caché, pero no crea una identidad de dispositivo ni un estado de emparejamiento iniciales.
    - El código de salida solo es distinto de cero cuando no se puede acceder a ninguno de los destinos sondeados.

  </Accordion>
  <Accordion title="Salida JSON">
    Nivel superior:

    - `ok`: al menos un destino es accesible.
    - `degraded`: al menos un destino aceptó una conexión, pero no completó los diagnósticos RPC detallados completos.
    - `capability`: mejor capacidad observada entre los destinos accesibles (`read_only`, `write_capable`, `admin_capable`, `pairing_pending`, `connected_no_operator_scope` o `unknown`).
    - `primaryTargetId`: mejor destino para tratarlo como el ganador activo, en este orden: URL explícita, túnel SSH, remoto configurado, bucle local.
    - `warnings[]`: registros de advertencia de mejor esfuerzo con `code`, `message` y `targetIds` opcional.
    - `network`: sugerencias de URL de bucle local/tailnet derivadas de la configuración actual y la red del host.
    - `discovery.timeoutMs` / `discovery.count`: el presupuesto de descubrimiento y el recuento de resultados reales utilizados para esta pasada de sondeo.

    Por destino (`targets[].connect`): `ok` (accesibilidad + clasificación degradada), `rpcOk` (éxito del RPC detallado completo), `scopeLimited` (el RPC detallado falló por falta del ámbito de operador).

    Por destino (`targets[].auth`): `role` y `scopes` se indican en `hello-ok` cuando están disponibles, junto con la clasificación `capability` mostrada.

  </Accordion>
  <Accordion title="Códigos de advertencia habituales">
    - `ssh_tunnel_failed`: falló la configuración del túnel SSH; el comando recurrió a sondeos directos.
    - `multiple_gateways`: se pudo acceder a identidades de Gateway distintas, o OpenClaw no pudo demostrar que los destinos accesibles correspondieran al mismo Gateway. Un túnel SSH, una URL de proxy o una URL remota configurada hacia el mismo Gateway no activa esta advertencia.
    - `auth_secretref_unresolved`: no se pudo resolver una SecretRef de autenticación configurada para un destino con errores.
    - `probe_scope_limited`: la conexión WebSocket se realizó correctamente, pero el sondeo de lectura estuvo limitado por la falta de `operator.read`.
    - `local_tls_runtime_unavailable`: TLS está habilitado en el Gateway local, pero OpenClaw no pudo cargar la huella digital del certificado local.

  </Accordion>
</AccordionGroup>

#### Acceso remoto mediante SSH (paridad con la aplicación para Mac)

El modo "Remote over SSH" de la aplicación para macOS utiliza un reenvío de puerto local para que un Gateway remoto limitado al bucle local sea accesible en `ws://127.0.0.1:<port>`.

Equivalente en la CLI:

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` o `user@host:port` (el puerto predeterminado es `22`).
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  Archivo de identidad.
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  Selecciona el primer host de Gateway descubierto como destino SSH a partir del punto de conexión de descubrimiento resuelto (`local.` más el dominio de área amplia configurado, si existe). Se ignoran las sugerencias procedentes únicamente de TXT.
</ParamField>

Valores predeterminados de configuración (opcionales): `gateway.remote.sshTarget`, `gateway.remote.sshIdentity`.

### `gateway call <method>`

Herramienta auxiliar RPC de bajo nivel.

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  Cadena de objeto JSON para los parámetros.
</ParamField>
<ParamField path="--url <url>" type="string">
  URL WebSocket del Gateway.
</ParamField>
<ParamField path="--token <token>" type="string">
  Token del Gateway.
</ParamField>
<ParamField path="--password <password>" type="string">
  Contraseña del Gateway.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  Presupuesto de tiempo de espera.
</ParamField>
<ParamField path="--expect-final" type="boolean">
  Principalmente para RPC de tipo agente que transmiten eventos intermedios antes de una carga final.
</ParamField>
<ParamField path="--json" type="boolean">
  Salida JSON legible por máquinas.
</ParamField>

<Note>
`--params` debe ser JSON válido, y cada método valida su propia estructura de parámetros (se rechazan los campos adicionales o con nombres incorrectos).
</Note>

## Gestionar el servicio del Gateway

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### Instalar con un contenedor ejecutable

Utilice `--wrapper` cuando el servicio gestionado deba iniciarse mediante otro ejecutable, por ejemplo, una capa de compatibilidad de gestor de secretos o una herramienta auxiliar para ejecutarlo como otro usuario. El contenedor recibe los argumentos normales del Gateway y es responsable de ejecutar finalmente mediante exec `openclaw` o Node con esos argumentos.

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

También puede establecer el contenedor mediante el entorno. `gateway install` valida que la ruta sea un archivo ejecutable, escribe el contenedor en el `ProgramArguments` del servicio y conserva `OPENCLAW_WRAPPER` en el entorno del servicio para posteriores reinstalaciones forzadas, actualizaciones y reparaciones del doctor.

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

Para eliminar un contenedor conservado, borre `OPENCLAW_WRAPPER` durante la reinstalación:

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="Opciones del comando">
    - `gateway status`: `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json`
    - `gateway install`: `--port`, `--runtime <node>` (valor predeterminado: `node`), `--token`, `--wrapper <path>`, `--force`, `--json`
    - `gateway restart`: `--safe`, `--skip-deferral`, `--force`, `--wait <duration>`, `--json`
    - `gateway uninstall|start`: `--json`
    - `gateway stop`: `--disable`, `--force`, `--json`

  </Accordion>
  <Accordion title="Comportamiento del ciclo de vida">
    - `gateway start` es idempotente: cuando el servicio gestionado ya está en ejecución, informa del proceso en ejecución y no lo modifica. Un servicio cargado pero detenido se inicia como antes.
    - Utilice `gateway restart` para reiniciar un servicio gestionado. No encadene `gateway stop` y `gateway start` como sustituto del reinicio.
    - En un shell no interactivo, `gateway stop` requiere `--force`. Los terminales interactivos mantienen el comportamiento existente sin solicitudes. Para automatización y pruebas, es preferible utilizar `gateway run --dev` o un `--profile` aislado con un puerto libre.
    - En macOS, `gateway stop` utiliza `launchctl bootout` de forma predeterminada, lo que elimina el LaunchAgent de la sesión de arranque actual sin conservar una desactivación: la recuperación automática de KeepAlive permanece activa para futuros fallos y `gateway start` lo vuelve a habilitar correctamente sin necesidad de ejecutar manualmente `launchctl enable`. Pase `--disable` para suprimir de forma persistente KeepAlive y RunAtLoad, de modo que el Gateway no vuelva a generarse hasta el siguiente `gateway start` explícito; utilice esta opción cuando una detención manual deba persistir tras los reinicios.
    - Las modificaciones del ciclo de vida del Gateway añaden registros de auditoría de clave-valor de mejor esfuerzo a `<state-dir>/logs/gateway-restart.log`, incluidas las operaciones de inicio, detención y reinicio de la CLI, las solicitudes de reinicio seguro, los reinicios del supervisor y las transferencias desvinculadas.
    - Los comandos del ciclo de vida aceptan `--json` para la creación de scripts.

  </Accordion>
  <Accordion title="Dimensionamiento del montón del Gateway gestionado">
    - `gateway install` escribe un valor `NODE_OPTIONS` exclusivo para el montón del servicio Gateway gestionado. Su objetivo es el 50 % de la memoria restringida cuando Node informa de un límite de contenedor o servicio; en caso contrario, el 50 % de la memoria física.
    - El intervalo objetivo nominal es de 2048–8192 MiB, con un límite adicional del 75 % para reservar espacio para la memoria nativa. En hosts pequeños, ese límite de espacio reservado puede situar el límite aplicado por debajo del mínimo nominal de 2048 MiB.
    - Un valor `--max-old-space-size` explícito y válido ya almacenado en el servicio instalado se conserva durante las reinstalaciones forzadas y las reparaciones del doctor. Otros indicadores de `NODE_OPTIONS` no se transfieren al servicio gestionado.
    - El valor `NODE_OPTIONS` del shell del entorno no anula esta política. Utilice `gateway status` o `doctor` para inspeccionar el valor instalado; ejecute `openclaw gateway install --force` para regenerar metadatos de servicios antiguos que no tengan una configuración de montón gestionado.
    - La política solo se aplica al servicio Gateway gestionado. `gateway run` en primer plano, los servicios de Node y las unidades del supervisor escritas manualmente conservan su propia configuración de tiempo de ejecución.

  </Accordion>
  <Accordion title="Autenticación y SecretRefs durante la instalación">
    - Cuando la autenticación mediante token requiere un token y `gateway.auth.token` se gestiona mediante SecretRef, `gateway install` valida que la SecretRef se pueda resolver, pero no conserva el token resuelto en los metadatos del entorno del servicio.
    - Si la autenticación mediante token requiere un token y la SecretRef del token configurada no se puede resolver, la instalación se cierra de forma segura en lugar de conservar texto sin formato de respaldo.
    - Para la autenticación mediante contraseña en `gateway run`, es preferible utilizar `OPENCLAW_GATEWAY_PASSWORD`, `--password-file` o un `gateway.auth.password` respaldado por SecretRef en lugar de `--password` insertado.
    - En el modo de autenticación inferido, `OPENCLAW_GATEWAY_PASSWORD` exclusivo del shell no relaja los requisitos de token de la instalación; utilice una configuración duradera (`gateway.auth.password` o la configuración `env`) al instalar un servicio gestionado.
    - Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está establecido, la instalación queda bloqueada hasta que se establezca explícitamente el modo.

  </Accordion>
</AccordionGroup>

## Descubrir gateways (Bonjour)

`gateway discover` busca balizas del Gateway (`_openclaw-gw._tcp`).

- DNS-SD multidifusión: `local.`
- DNS-SD unidifusión (Bonjour de área amplia): elija un dominio (por ejemplo, `openclaw.internal.`) y configure DNS dividido y un servidor DNS; consulte [Bonjour](/es/gateway/bonjour).

Solo anuncian la baliza los gateways que tienen habilitado el descubrimiento mediante Bonjour (valor predeterminado).

Sugerencias TXT en cada baliza: `role` (sugerencia de función del Gateway), `transport` (sugerencia de transporte, p. ej., `gateway`), `gatewayPort` (puerto WebSocket, normalmente `18789`), `tailnetDns` (nombre de host de MagicDNS, cuando está disponible), `gatewayTls` / `gatewayTlsSha256` (TLS habilitado + huella digital del certificado). `sshPort` y `cliPath` se publican únicamente en el modo de descubrimiento completo (`discovery.mdns.mode: "full"`; el valor predeterminado es `"minimal"`, que los omite; en ese caso, los clientes utilizan de forma predeterminada el puerto `22` para los destinos SSH).

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  Tiempo de espera por comando (exploración/resolución).
</ParamField>
<ParamField path="--json" type="boolean">
  Salida legible por máquinas (también deshabilita los estilos y el indicador de carga).
</ParamField>

Ejemplos:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- Busca en `local.` y en el dominio de área amplia configurado cuando hay uno habilitado.
- `wsUrl` en la salida JSON se deriva del punto de conexión de servicio resuelto, no de sugerencias procedentes únicamente de TXT, como `lanHost` o `tailnetDns`.
- `discovery.mdns.mode` controla la publicación de `sshPort`/`cliPath` tanto en mDNS `local.` como en DNS-SD de área amplia (véase más arriba).

</Note>

## Temas relacionados

- [Referencia de la CLI](/es/cli)
- [Guía operativa del Gateway](/es/gateway)
