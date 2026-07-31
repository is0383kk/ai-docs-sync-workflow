---
read_when:
    - Necesita responder quién ejecutó un agente o una herramienta, cuándo se ejecutó y cómo finalizó.
    - Necesita metadatos del ciclo de vida de los mensajes entrantes o salientes sin contenido.
    - Necesita una exportación de actividad acotada y segura para la censura de datos sensibles.
summary: Referencia de la CLI para registros de auditoría de metadatos del ciclo de vida de ejecuciones, herramientas y mensajes
title: Registros de auditoría
x-i18n:
    generated_at: "2026-07-26T05:02:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da9df6f388b0a24c3b79d755fa59d047cce99262bc6d9c890be7a83da75693a8
    source_path: cli/audit.md
    workflow: 16
---

# `openclaw audit`

Consulta el registro de auditoría de solo metadatos del Gateway para ejecuciones de agentes, acciones de herramientas y
registros opcionales del ciclo de vida de los mensajes.

El registro está activado de forma predeterminada para los eventos de ejecución y herramientas. Establece
[`audit.enabled: false`](/es/gateway/configuration-reference#audit) y reinicia el
Gateway para detener todos los registros de eventos nuevos. Los registros de mensajes están desactivados por separado de forma
predeterminada; establece `audit.messages` en `direct` o `all` y reinicia el Gateway para
registrarlos. Los registros existentes se pueden consultar hasta que caduquen (30 días).

El registro es independiente de las transcripciones de conversaciones: registra identidad,
orden, procedencia, acción, estado y códigos de resultado normalizados, pero nunca
almacena contenido, y los identificadores de mensajes aparecen únicamente como seudónimos con clave
locales de la instalación. El [historial de auditoría](/es/gateway/audit) define el modelo de datos completo,
la semántica de privacidad, los límites de almacenamiento y retención y las limitaciones de cobertura; esta página
abarca la interfaz de comandos.

```bash
openclaw audit
openclaw audit --agent main --status failed
openclaw audit --session "agent:main:main" --after 2026-07-01T00:00:00Z
openclaw audit --run 8c69f72e-8b11-4c54-98d5-1a3dd67450c3
openclaw audit --kind tool_action --limit 50 --json
openclaw audit --kind message --direction outbound --channel telegram --json
```

## Filtros

- `--agent <id>`: id exacto del agente
- `--session <key>`: clave exacta de la sesión
- `--run <id>`: id exacto de la ejecución
- `--kind <kind>`: `agent_run`, `tool_action` o `message`
- `--status <status>`: `started`, `succeeded`, `failed`, `cancelled`,
  `timed_out`, `blocked` o `unknown`
- `--direction <direction>`: dirección del mensaje, `inbound` o `outbound`
- `--channel <channel>`: canal exacto del mensaje
- `--after <timestamp>` / `--before <timestamp>`: marca de tiempo ISO inclusiva o
  milisegundos Unix
- `--limit <count>`: tamaño de página de 1 a 500; valor predeterminado `100`
- `--cursor <sequence>`: continúa una consulta anterior ordenada de más reciente a más antiguo
- `--json`: imprime la página acotada como JSON

La CLI consulta el RPC de actividad versionado para que un solo comando muestre el registro
configurado completo. La salida de texto muestra la hora, el tipo, la dirección, el canal, el estado,
el agente, la ejecución y la acción. La procedencia ausente de un mensaje se representa como `-`; OpenClaw
no inventa ids de agentes ni de ejecuciones. Las acciones de herramientas también muestran el nombre de la herramienta. La salida
JSON incluye `nextCursor` cuando existe otra página. Pasa ese valor a
`--cursor` para continuar sin reordenar los registros que lleguen durante la paginación.

Estas exportaciones siguen siendo metadatos operativos sensibles aunque no incluyan los cuerpos de los mensajes
ni los campos de identidad sin procesar de los mensajes. Los ids de agentes, sesiones y ejecuciones, los tiempos,
los canales, los resultados y las referencias HMAC estables pueden correlacionar la actividad. Protégelos
con los mismos controles de acceso y prácticas de retención que los demás registros
del operador.

## Eventos registrados

El Gateway proyecta flujos de ciclo de vida fiables en seis acciones:

- `agent.run.started`
- `agent.run.finished`
- `tool.action.started`
- `tool.action.finished`
- `message.inbound.processed`
- `message.outbound.finished`

Cada registro devuelto tiene un id de evento estable, una secuencia del registro
que aumenta de forma monotónica, una marca de tiempo del ciclo de vida, actor, acción, estado, un
marcador `schemaVersion: 1`, secuencia de origen y `redaction: "metadata_only"`.
La procedencia del agente, la sesión y la ejecución, así como los campos específicos del evento, solo están presentes cuando
la fuente fiable los proporciona. Los registros de mensajes omiten intencionadamente
`sessionKey` y `sessionId`, por lo que los filtros `--session` solo se aplican a registros de ejecuciones y herramientas.

Los registros terminales de ejecuciones y herramientas distinguen entre éxito, fallo, cancelación,
tiempo de espera agotado y bloqueos de políticas mediante estados cerrados y códigos de error. `unknown` es un
resultado explícito distinto del éxito cuando un entorno de ejecución ascendente no expone un
resultado terminal autoritativo. Los ids de llamadas a herramientas solo se exportan como
huellas digitales estables. Los nombres de herramientas deben coincidir con el contrato compacto de nombres
orientado al modelo; los demás valores se convierten en `unknown`.

Los registros de mensajes añaden la dirección, el canal, el tipo de conversación, el resultado y,
opcionalmente, el tipo de entrega, la etapa del fallo, la duración, el recuento de resultados, el código
de motivo normalizado y seudónimos con clave de cuenta, conversación, mensaje y destino. El
límite de entrada actual abarca los mensajes aceptados que llegan al despacho central,
incluidos los resultados de duplicación y procesamiento terminal del núcleo. El límite de salida
escribe una fila terminal por cada carga útil original de respuesta lógica que llega
a la entrega duradera compartida; la fragmentación y la distribución del adaptador se agregan en
`resultCount`. Los envíos en cola reintentables o ambiguos solo se registran después de que una
confirmación, una cola de mensajes no entregados o una reconciliación haga que el resultado sea terminal.
Las rutas locales del Plugin y de envío directo que eluden esos límites compartidos todavía no
están cubiertas; la ausencia de una fila no demuestra que no existiera ningún mensaje.

El registro de auditoría no sustituye las transcripciones, el historial de tareas, el historial de ejecuciones de Cron
ni los registros. Proporciona un pequeño índice entre ejecuciones para las consultas del operador sin
copiar el contenido de las conversaciones en otro almacén.

Para las filas de entrada, `durationMs` mide el despacho central y `resultCount` cuenta
las cargas útiles finalizadas en cola de herramientas, bloqueos y respuestas. Para las filas de salida,
`durationMs` incluye la propiedad de la entrega hasta su estado terminal (y, por tanto,
el tiempo de espera en cola), mientras que `resultCount` cuenta los envíos físicos identificados de la
plataforma. `deliveryKind`, cuando está presente, describe la carga útil efectiva posterior al hook
y al renderizado; las filas suprimidas y ambiguas por fallos omiten este valor.

## RPC del Gateway

`audit.activity.list` requiere `operator.read` y acepta los mismos filtros. Devuelve
la unión de eventos de actividad V1 con nombre, incluidos registros de ejecuciones, herramientas, mensajes entrantes
y mensajes salientes.

```bash
openclaw gateway call audit.activity.list --params '{"channel":"telegram","limit":50}'
```

El resultado es `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`.
Los resultados se ordenan de más reciente a más antiguo y se limitan a 500 registros por solicitud.

El RPC `audit.list` distribuido permanece sin cambios para los clientes antiguos de ejecuciones y herramientas. Cuando
`audit.activity.list` no está disponible en un Gateway antiguo, la CLI vuelve a intentar
`audit.list` solo si el método heredado admite todos los filtros solicitados. `--kind message`,
`--direction` y `--channel` fallan con un mensaje de actualización en un Gateway antiguo
en lugar de descartarse de forma silenciosa.

## Contenido relacionado

- [Historial de auditoría](/es/gateway/audit)
- [Protocolo del Gateway](/es/gateway/protocol#audit-ledger-rpc)
- [Sesiones](/es/cli/sessions)
- [Tareas](/es/cli/tasks)
- [Trabajos de Cron](/es/automation/cron-jobs)
