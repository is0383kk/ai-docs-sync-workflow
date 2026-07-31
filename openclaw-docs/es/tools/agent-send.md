---
read_when:
    - Quieres activar ejecuciones del agente desde scripts o desde la línea de comandos.
    - Necesita enviar mediante programación las respuestas del agente a un canal de chat.
summary: Ejecuta turnos del agente desde la CLI y, opcionalmente, envía las respuestas a los canales
title: Envío del agente
x-i18n:
    generated_at: "2026-07-26T04:52:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3da0feea102725ebb5555e0dd375ed6f3a0396d8ffd0ab916ced303201eabc
    source_path: tools/agent-send.md
    workflow: 16
---

`openclaw agent` ejecuta un único turno del agente desde la línea de comandos sin un
mensaje de chat entrante. Úselo para flujos de trabajo con scripts, pruebas y
entrega programática. Referencia completa de indicadores y comportamiento:
[Referencia de la CLI del agente](/es/cli/agent).

## Inicio rápido

<Steps>
  <Step title="Ejecutar un turno sencillo del agente">
    ```bash
    openclaw agent --agent main --message "¿Qué tiempo hace hoy?"
    ```

    Envía el mensaje a través del Gateway e imprime la respuesta.

  </Step>

  <Step title="Enviar una instrucción multilínea desde un archivo">
    ```bash
    openclaw agent --agent ops --message-file ./task.md
    ```

    Lee un archivo UTF-8 válido como cuerpo del mensaje del agente.

  </Step>

  <Step title="Seleccionar un agente o una sesión específicos">
    ```bash
    # Seleccionar un agente específico
    openclaw agent --agent ops --message "Resume los registros"

    # Seleccionar un número de teléfono (deriva la clave de sesión)
    openclaw agent --to +15555550123 --message "Actualización de estado"

    # Reutilizar una sesión existente
    openclaw agent --session-id abc123 --message "Continúa la tarea"

    # Seleccionar una clave de sesión exacta
    openclaw agent --session-key agent:ops:incident-42 --message "Resume el estado"
    ```

  </Step>

  <Step title="Entregar la respuesta a un canal">
    ```bash
    # Entregar a WhatsApp (canal predeterminado)
    openclaw agent --to +15555550123 --message "Informe listo" --deliver

    # Entregar a Slack
    openclaw agent --agent ops --message "Genera un informe" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

  </Step>
</Steps>

## Indicadores

| Indicador                   | Descripción                                                          |
| --------------------------- | -------------------------------------------------------------------- |
| `--message <text>`          | Mensaje en línea que se enviará                                      |
| `--message-file <path>`     | Leer el mensaje desde un archivo UTF-8 válido (máx. 4 MiB)           |
| `--to <dest>`               | Derivar la clave de sesión de un destino (teléfono, id de chat)      |
| `--session-key <key>`       | Usar una clave de sesión explícita                                   |
| `--agent <id>`              | Seleccionar un agente configurado (usa su sesión `main`) |
| `--session-id <id>`         | Reutilizar una sesión existente mediante su id                       |
| `--model <id>`              | Sustitución del modelo para esta ejecución (`provider/model` o id del modelo) |
| `--local`                   | Forzar el entorno de ejecución integrado local (omitir el Gateway)   |
| `--deliver`                 | Enviar la respuesta a un canal de chat                               |
| `--channel <name>`          | Canal de entrega; con `--agent` + `--to`, también se aplica al ámbito de los mensajes directos |
| `--reply-to <target>`       | Sustitución del destino de entrega                                   |
| `--reply-channel <name>`    | Sustitución del canal de entrega                                     |
| `--reply-account <id>`      | Sustitución del id de la cuenta de entrega                           |
| `--thinking <level>`        | Establecer el nivel de razonamiento del perfil de modelo seleccionado |
| `--verbose <on\|full\|off>` | Conservar el nivel de detalle de la sesión (`full` también registra la salida de las herramientas) |
| `--timeout <seconds>`       | Sustituir el tiempo de espera del agente (valor predeterminado: 600, o el valor de configuración) |
| `--json`                    | Generar una salida JSON estructurada                                 |

## Comportamiento

- De forma predeterminada, la CLI funciona **a través del Gateway**. Añada `--local` para forzar el
  entorno de ejecución integrado en la máquina actual.
- Proporcione exactamente uno de `--message` o `--message-file`. Los mensajes de archivo conservan
  el contenido multilínea después de eliminar una marca BOM UTF-8 opcional. Los archivos de más de
  4 MiB se rechazan antes del envío.
- Tras reintentos transitorios del protocolo de enlace, un tiempo de espera agotado del Gateway o una conexión cerrada
  hacen que el comando falle con una indicación en stderr; la CLI nunca vuelve a ejecutar silenciosamente el turno
  de forma integrada. Es posible que el Gateway finalice aun así un turno aceptado, por lo que se deben verificar el Gateway
  y el estado de la sesión antes de volver a intentarlo o ejecutarlo con `--local`.
- Selección de sesión: `--to` deriva la clave de sesión (los destinos de grupo o canal
  mantienen el aislamiento; los chats directos se agrupan en `main`). Cuando se usan conjuntamente `--agent`,
  `--channel` y `--to`, el enrutamiento sigue el destinatario canónico
  y `session.dmScope` del canal. Las identidades estables exclusivamente de salida usan una
  sesión propiedad del proveedor aislada de la sesión principal del agente.
- `--session-key` selecciona una clave explícita. Las claves con prefijo de agente deben usar
  `agent:<agent-id>:<session-key>`, y `--agent` debe coincidir con ese id de agente cuando
  se proporcionan ambos. Las claves simples que no sean centinelas se limitan al ámbito de `--agent` cuando
  se proporciona; por ejemplo, `--agent ops --session-key incident-42` se dirige a
  `agent:ops:incident-42`. Sin `--agent`, las claves simples que no sean centinelas se limitan al ámbito
  del agente predeterminado configurado. Los valores literales `global` y `unknown` permanecen
  sin ámbito solo cuando no se proporciona `--agent`.
- `--reply-channel` y `--reply-account` solo afectan a la entrega.
- Los indicadores de razonamiento y nivel de detalle se conservan en el almacén de sesiones.
- Salida: texto sin formato de manera predeterminada, o `--json` para obtener una carga estructurada y metadatos.
- Con `--json --deliver`, el JSON incluye el estado de entrega de los envíos
  realizados, suprimidos, parciales y fallidos. Consulte
  [Estado de entrega JSON](/es/cli/agent#json-delivery-status).

## Ejemplos

```bash
# Turno sencillo con salida JSON
openclaw agent --to +15555550123 --message "Rastrea los registros" --verbose on --json

# Turno con sustitución del modelo
openclaw agent --agent ops --model openai/gpt-5.4 --message "Resume los registros"

# Turno con nivel de razonamiento
openclaw agent --session-id 1234 --message "Resume la bandeja de entrada" --thinking medium

# Instrucción multilínea desde un archivo
openclaw agent --agent ops --message-file ./task.md

# Clave de sesión exacta
openclaw agent --session-key agent:ops:incident-42 --message "Resume el estado"

# Clave heredada limitada al ámbito de un agente
openclaw agent --agent ops --session-key incident-42 --message "Resume el estado"

# Entregar a un canal distinto del de la sesión
openclaw agent --agent ops --message "Alerta" --deliver --reply-channel telegram --reply-to "@admin"
```

## Temas relacionados

<CardGroup cols={2}>
  <Card title="Referencia de la CLI del agente" href="/es/cli/agent" icon="terminal">
    Referencia completa de los indicadores y las opciones de `openclaw agent`.
  </Card>
  <Card title="Subagentes" href="/es/tools/subagents" icon="users">
    Creación de subagentes en segundo plano.
  </Card>
  <Card title="Sesiones" href="/es/concepts/session" icon="comments">
    Cómo funcionan las claves de sesión y cómo las resuelven `--to`, `--agent` y `--session-id`.
  </Card>
  <Card title="Comandos de barra diagonal" href="/es/tools/slash-commands" icon="slash">
    Catálogo de comandos nativos utilizado dentro de las sesiones de agente.
  </Card>
</CardGroup>
