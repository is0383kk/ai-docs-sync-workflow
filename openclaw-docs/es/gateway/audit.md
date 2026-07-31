---
read_when:
    - Se necesita un registro duradero de lo que hizo el Gateway sin almacenar contenido
    - Está decidiendo si habilitar la auditoría del ciclo de vida de los mensajes
    - Debe explicar qué demuestran y qué no demuestran los registros de auditoría
summary: Historial de auditoría solo de metadatos para ejecuciones de agentes, acciones de herramientas y ciclos de vida de mensajes opcionales
title: Historial de auditoría
x-i18n:
    generated_at: "2026-07-26T05:10:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1005b214a674f0f888d759837bd627be458cefcf9ed61bda722499333361dc45
    source_path: gateway/audit.md
    workflow: 16
---

# Historial de auditoría

El Gateway mantiene un registro de auditoría acotado y compuesto únicamente por metadatos en la base de datos de estado compartida de OpenClaw. Permite responder preguntas operativas como «qué agente se ejecutó, cuándo y cómo terminó», «qué acciones de herramientas ejecutó una ejecución» y, cuando la auditoría de mensajes está habilitada, «si un mensaje entrante aceptado llegó al despacho» y «si un mensaje saliente alcanzó un estado terminal de entrega».

El registro almacena identidad, orden, procedencia, acción, estado y códigos de resultado normalizados. Nunca almacena prompts, cuerpos de mensajes, argumentos de herramientas, resultados de herramientas, archivos adjuntos, nombres de archivo, URLs, salida de comandos ni texto de error sin procesar.

## Familias de registros

Los eventos de ejecuciones y herramientas se registran siempre que la auditoría está habilitada (de forma predeterminada). Los eventos del ciclo de vida de los mensajes son opcionales y están deshabilitados de forma predeterminada.

| Familia                 | Acciones                                                 | Valor predeterminado |
| ----------------------- | -------------------------------------------------------- | -------------------- |
| Ejecuciones de agentes  | `agent.run.started`, `agent.run.finished`                   | activado             |
| Acciones de herramientas | `tool.action.started`, `tool.action.finished`                  | activado             |
| Mensajes                | `message.inbound.processed`, `message.outbound.finished`                   | desactivado          |

Cada registro contiene un identificador de evento estable, una secuencia monotónica del registro, una marca de tiempo del ciclo de vida, un actor, una acción, un estado, `schemaVersion: 1` y `redaction: "metadata_only"`. Consulte [Registros de auditoría](/es/cli/audit) para ver la referencia completa de los campos y los filtros de consulta.

## Eventos del ciclo de vida de los mensajes

Establezca [`audit.messages`](/es/gateway/configuration-reference#audit) para elegir qué se registra y, a continuación, reinicie el Gateway:

- `off` (valor predeterminado): no se registran mensajes.
- `direct`: solo mensajes de conversaciones directas.
- `all`: mensajes directos, de grupo y de canal.

Dos límites autoritativos generan registros de mensajes:

- Las filas **entrantes** se escriben cuando un mensaje aceptado llega al despacho del núcleo, incluidos los resultados de procesamiento duplicados y terminales.
- Las filas **salientes** se escriben cuando la entrega duradera compartida alcanza un resultado terminal: enviado, suprimido, fallido o un `unknown` explícito para envíos ambiguos debido a un fallo. Se incluyen la recuperación de la cola y los resultados de mensajes no entregables. Cada carga útil original de respuesta lógica obtiene una fila terminal; la fragmentación y la distribución en abanico de adaptadores se agregan en `resultCount`.

### Clasificación del tipo de conversación

El modo `direct` constituye un límite de privacidad, por lo que un mensaje se clasifica como conversación directa únicamente cuando los datos del destino lo demuestran: la ruta de envío declaró el tipo de conversación de destino, o la ruta de la sesión de entrega nombra exactamente el canal y el interlocutor al que se realiza la entrega. Las señales más débiles, como el estado de las políticas o la conversación de origen, pueden clasificar un mensaje como `group` (excluyéndolo de la recopilación `direct`), pero nunca pueden afirmar `direct`. Los mensajes que no pueden demostrarse como directos se clasifican como `unknown` y no se registran en el modo `direct`. Por lo tanto, los canales que no declaran tipos de chat pueden registrar menos filas en el modo `direct` que en el modo `all`.

## Modelo de privacidad

Las filas de mensajes nunca almacenan identificadores de plataforma sin procesar. Los identificadores de cuenta, conversación, mensaje y destino, cuando la correlación está disponible, se exportan únicamente como seudónimos con clave locales de la instalación (`hmac-sha256:v1:<keyId>:<digest>`):

- La clave HMAC se genera durante el primer uso, está separada por dominio para cada tipo de identificador y reside en la misma base de datos de estado que el registro.
- Los seudónimos son estables dentro de una instalación, por lo que las filas relativas a una misma conversación pueden correlacionarse sin revelar el identificador de la plataforma.
- Esto es **correlación, no anonimización**: cualquier persona con acceso de lectura a la base de datos de estado también dispone de la clave y puede comprobar identificadores sin procesar candidatos frente a los seudónimos. Las exportaciones de RPC y CLI nunca incluyen la clave.
- Si el material de la clave falta o está dañado mientras se conservan filas de mensajes, el Gateway adopta un cierre seguro y descarta los nuevos registros de mensajes en lugar de rotar silenciosamente a una clave nueva, lo que dividiría la correlación.

Los registros de ejecuciones y herramientas conservan `sessionKey` y `sessionId` para la correlación; las claves de sesión canónicas pueden contener identificadores de cuentas o interlocutores de la plataforma. Los registros de mensajes omiten ambos de manera intencionada.

Las exportaciones de auditoría siguen siendo metadatos operativos sensibles incluso sin contenido: los tiempos, los canales, los resultados y los seudónimos estables pueden correlacionar la actividad. Proteja las exportaciones con los mismos controles de acceso y prácticas de retención que los demás registros del operador.

## Límites de cobertura y demostración

El registro funciona con el mejor esfuerzo y está deliberadamente acotado. Trátelo como evidencia de lo que se registró, no como demostración de lo que ocurrió:

- **La ausencia de una fila no demuestra nada.** Los descartes entrantes previos a la admisión, los envíos desde procesos de la CLI sin un registrador del Gateway en ejecución y las rutas locales del Plugin o de envío directo que omiten la entrega duradera compartida no dejan ningún registro.
- Las escrituras pasan por un proceso de trabajo en segundo plano acotado; un fallo del proceso de trabajo o la saturación de la cola descartan registros y generan una advertencia operativa.
- Los envíos salientes ambiguos debido a un fallo se registran como `unknown` en lugar de asignarles resultados inventados.

Este registro facilita la depuración y la revisión operativa. No es un archivo de cumplimiento sin pérdidas; si se necesita uno, debe utilizarse un sistema externo alimentado por [OpenTelemetry](/es/gateway/opentelemetry) o herramientas del canal.

## Almacenamiento, retención y migración

Los registros residen en la base de datos de estado compartida (`state/openclaw.sqlite`) y se escriben fuera de la ruta crítica de entrega. Las consultas nunca devuelven registros con más de 30 días de antigüedad y el registro tiene un límite de 100,000 filas; las filas vencidas se eliminan durante el inicio, el mantenimiento horario y las escrituras posteriores. El mantenimiento de la retención continúa ejecutándose incluso cuando la recopilación está deshabilitada.

Al actualizar desde un Gateway con el registro anterior limitado a ejecuciones y herramientas, el esquema se migra automáticamente durante el inicio (o mediante `openclaw doctor --fix`); se conservan las filas existentes y sus secuencias del registro.

## Consultas

- CLI: [`openclaw audit`](/es/cli/audit) con filtros por agente, sesión, ejecución, tipo, estado, dirección, canal, límites temporales y paginación por cursor.
- RPC del Gateway: `audit.activity.list` (requiere `operator.read`) devuelve la unión versionada V1 de eventos de actividad; la RPC `audit.list` distribuida permanece sin cambios para los clientes anteriores de ejecuciones y herramientas. Consulte [Protocolo del Gateway](/es/gateway/protocol#audit-ledger-rpc).

## Relacionado

- [CLI de registros de auditoría](/es/cli/audit)
- [Referencia de configuración](/es/gateway/configuration-reference#audit)
- [Protocolo del Gateway](/es/gateway/protocol#audit-ledger-rpc)
- [OpenTelemetry](/es/gateway/opentelemetry)
