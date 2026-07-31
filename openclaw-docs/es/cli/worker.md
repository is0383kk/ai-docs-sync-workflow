---
read_when:
    - Operación o depuración de trabajadores en la nube iniciados por el Gateway
    - Verificación de la admisión de workers, la asignación de sesiones o el aislamiento de herramientas locales
summary: Referencia interna para operadores del entorno de ejecución restringido de trabajadores en la nube
title: Trabajador
x-i18n:
    generated_at: "2026-07-26T04:35:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c4749e2abaf4fca00d903114b0661454d67207547fe17711dc5315656e0cd14
    source_path: cli/worker.md
    workflow: 16
---

# `openclaw worker`

`openclaw worker` es el punto de entrada de ejecución restringido que un
orquestador de trabajadores en la nube inicia dentro de un entorno de trabajador
preparado. No es un comando de uso general para registrar trabajadores manualmente.

El Gateway instala el paquete de OpenClaw correspondiente y abre el túnel SSH inverso
con la clave del host fijada. El iniciador del trabajador ejecuta este comando con una
asignación preparada. El comando se conecta a través del socket local reenviado por el
túnel y se admite con el rol dedicado `worker`.

## Contrato de inicio

El comando lee exactamente un sobre de inicio JSON acotado desde la entrada estándar.
El sobre contiene la ubicación del socket local, la credencial emitida del trabajador,
la identidad del paquete y del protocolo, la época del propietario, la única sesión y
el único turno asignados, y los nombres exactos de las herramientas locales del
trabajador autorizadas para ese turno. El Gateway resuelve este conjunto final de
herramientas a partir de la política vigente antes de la transferencia; la configuración
sin procesar y la identidad del propietario programado nunca entran en el sobre del
trabajador.
La credencial nunca se acepta mediante argumentos de línea de comandos, y esta página
no proporciona intencionadamente ningún ejemplo de credencial ni de sobre elaborado
manualmente.

La admisión se deniega de forma segura si el sobre no es válido, se rechaza la
credencial, las características del paquete o del protocolo no coinciden, o la sesión
y la época del propietario ya no están vigentes. Los nombres de herramientas ausentes,
duplicados o desconocidos también invalidan el sobre. Los operadores deben iniciar los
trabajadores mediante el orquestador de trabajadores en la nube en lugar de invocar
directamente este punto de entrada.

## Límite de ejecución

El proceso ejecuta el bucle normal del agente integrado con un backend restringido:

- Las herramientas de programación `read`, `write`, `edit`, `apply_patch`, `exec` y `process`
  se ejecutan localmente en el espacio de trabajo del trabajador cuando están presentes
  en la autoridad de turno emitida por el Gateway. Una autoridad vacía ejecuta el modelo
  sin herramientas.
- Las llamadas al modelo utilizan el proxy de inferencia del Gateway. No se carga
  ningún perfil local de autenticación del modelo.
- Las escrituras de la transcripción utilizan la RPC de confirmación de
  transcripciones del Gateway.
- La transmisión y las actualizaciones del ciclo de vida de las herramientas
  utilizan la RPC de eventos en directo del Gateway.
- Solo se aceptan la sesión y el turno asignados.

El modo de trabajador no inicia canales, superficies HTTP del Gateway ni el inicio
automático de plugins más allá del conjunto de herramientas de la sesión asignada.
Utiliza un directorio de estado desechable y no dispone de credenciales permanentes
de proveedor ni de plataforma de desarrollo.

El envío de sesiones entre trabajadores no está expuesto en este modo. La ubicación
y el envío siguen siendo propiedad del Gateway: un operador puede enviar mediante el
Gateway una sesión local existente de un árbol de trabajo administrado, mientras que
un proceso de trabajador no puede enviarse a sí mismo ni enviar a otro trabajador.

La asignación preparada contiene el contexto de la transcripción, la hoja base
aceptada, la secuencia de confirmaciones y el cursor de eventos en directo. Cuando se
vuelve a conectar el túnel, el proceso se readmite con la misma credencial y época del
propietario, conserva la base aceptada de la transcripción, reproduce la cola de eventos
en directo que no se ha confirmado y vuelve a adjuntarse a un turno de inferencia en
curso con la misma identidad. El mensaje de inferencia terminal es la fuente de
autoridad si se perdieron deltas transmitidos. Una época del propietario que sustituya
a la anterior aísla el proceso y provoca una salida limpia.

Un rechazo de transcripción `stale-base-leaf` detiene de inmediato la ejecución
actual. El modo de trabajador no reintenta la secuencia rechazada contra una hoja
diferente, por lo que no se genera ninguna confirmación duplicada; se pierde cualquier
cola en memoria de esa ejecución que todavía no se haya confirmado. El reinicio
corresponde al propietario de ubicación del hito 3, que debe crear una asignación nueva
a partir de la transcripción y el registro de confirmaciones autoritativos del Gateway.
Asimismo, el reinicio de un proceso del Gateway termina un turno de inferencia pendiente
con un error del proveedor; solo la reconexión del túnel o del WebSocket del trabajador
puede volver a adjuntarse a un flujo de inferencia activo del mismo proceso.

Consulte [Protocolo del Gateway](/es/gateway/protocol#worker-role-and-closed-protocol) para
conocer la superficie RPC cerrada del trabajador y el [Plan de trabajadores en la nube](/es/plan/cloud-workers)
para conocer la arquitectura y el modelo de seguridad.
