---
read_when:
    - Explicación de cómo se comporta steer mientras un agente utiliza herramientas
    - Cambio del comportamiento de la cola de ejecuciones activas o de la integración de la dirección del entorno de ejecución
    - Comparación del modo de dirección con los modos de cola de seguimiento, recopilación e interrupción
summary: Cómo el direccionamiento de ejecuciones activas pone en cola los mensajes en los límites del entorno de ejecución
title: Cola de direccionamiento
x-i18n:
    generated_at: "2026-07-26T05:11:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 131f04f19934b9b1f6dd8ffb2cf2428950c319483abdc2ccdecec741809cda2a
    source_path: concepts/queue-steering.md
    workflow: 16
---

Cuando llega un prompt normal mientras la ejecución de una sesión ya está transmitiendo y el modo de cola es `steer` (el predeterminado, no requiere configuración), OpenClaw intenta enviar ese prompt al entorno de ejecución activo. OpenClaw y el arnés nativo del servidor de aplicaciones de Codex implementan los detalles de entrega de forma diferente.

Esta página aborda la redirección del modo de cola para mensajes entrantes normales en el modo `steer`. En el modo `followup` o `collect`, los mensajes normales omiten esta ruta y esperan hasta que finalice la ejecución activa. Para el comando explícito `/steer <message>`, consulte [Redirigir](/es/tools/steer).

## Límite del entorno de ejecución

La redirección no interrumpe una llamada a una herramienta que ya está en curso. OpenClaw comprueba si hay mensajes de redirección en cola en los límites del modelo:

1. El asistente solicita llamadas a herramientas.
2. OpenClaw ejecuta el lote de llamadas a herramientas del mensaje actual del asistente.
3. OpenClaw emite el evento de fin de turno.
4. OpenClaw vacía los mensajes de redirección en cola.
5. OpenClaw añade esos mensajes como mensajes del usuario antes de la siguiente llamada al LLM.

Esto mantiene los resultados de las herramientas asociados al mensaje del asistente que los solicitó y, a continuación, permite que la siguiente llamada al modelo vea la entrada más reciente del usuario.

El arnés nativo del servidor de aplicaciones de Codex expone `turn/steer` en lugar de la cola de redirección interna del entorno de ejecución de OpenClaw. OpenClaw agrupa los prompts en cola durante el intervalo de inactividad configurado y, a continuación, envía una única solicitud `turn/steer` con todas las entradas de usuario recopiladas en orden de llegada.

Los turnos de revisión y Compaction de Codex rechazan la redirección durante el mismo turno. Cuando un entorno de ejecución no puede aceptar la redirección en el modo `steer`, OpenClaw espera a que finalice la ejecución activa antes de iniciar el prompt.

## Modos

| Modo        | Comportamiento durante la ejecución activa                         | Comportamiento posterior                                                                       |
| ----------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| `steer`     | Redirige el prompt al entorno de ejecución activo cuando es posible. | Espera a que finalice la ejecución activa si la redirección no está disponible.                |
| `followup`  | No redirige.                                                       | Ejecuta los mensajes en cola más adelante, después de que termine la ejecución activa.          |
| `collect`   | No redirige.                                                       | Combina los mensajes en cola compatibles en un turno posterior tras el intervalo de antirrebote. |
| `interrupt` | Cancela la ejecución activa en lugar de redirigirla.               | Inicia el mensaje más reciente después de cancelarla.                                           |

## Ejemplo de ráfaga

Si cuatro usuarios envían mensajes mientras el agente ejecuta una llamada a una herramienta:

- Con el comportamiento predeterminado, el entorno de ejecución activo recibe los cuatro mensajes en orden de llegada antes de su siguiente decisión del modelo. OpenClaw los vacía en el siguiente límite del modelo; Codex los recibe como un único `turn/steer` agrupado.
- Con `/queue collect`, OpenClaw no redirige. Espera hasta que finalice la ejecución activa y, a continuación, crea un turno de seguimiento con los mensajes en cola compatibles tras el intervalo de antirrebote.
- Con `/queue interrupt`, OpenClaw cancela la ejecución activa e inicia el mensaje más reciente en lugar de redirigirlo.

## Alcance

La redirección siempre se dirige a la ejecución activa de la sesión actual. No crea una sesión nueva, no cambia la política de herramientas de la ejecución activa ni divide los mensajes por remitente. En los canales multiusuario, los prompts entrantes ya incluyen el contexto del remitente y de la ruta, por lo que la siguiente llamada al modelo puede ver quién envió cada mensaje.

Utilice `followup` o `collect` cuando quiera que los mensajes se pongan en cola de forma predeterminada en lugar de redirigirse a la ejecución activa. Utilice `interrupt` cuando el prompt más reciente deba sustituir la ejecución activa.

## Antirrebote

El antirrebote de cola integrado se aplica a la entrega en cola de `followup` y `collect`. En el modo `steer` con el arnés nativo de Codex, también establece el intervalo de inactividad antes de enviar `turn/steer` agrupados. En OpenClaw, la redirección activa en sí no utiliza el temporizador de antirrebote porque OpenClaw agrupa los mensajes de forma natural hasta el siguiente límite del modelo.

## Contenido relacionado

- [Cola de comandos](/es/concepts/queue)
- [Redirigir](/es/tools/steer)
- [Mensajes](/es/concepts/messages)
- [Bucle del agente](/es/concepts/agent-loop)
