---
read_when:
    - Quieres entender qué significa «contexto» en OpenClaw
    - Se está depurando por qué el modelo «sabe» algo (o lo olvidó)
    - Quiere reducir la sobrecarga de contexto (/context, /status, /compact)
summary: 'Contexto: qué ve el modelo, cómo se construye y cómo inspeccionarlo'
title: Contexto
x-i18n:
    generated_at: "2026-07-26T05:10:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1eb3d342a601a447487640587f746cc80a133ede338a880741f53c3e01f20ed1
    source_path: concepts/context.md
    workflow: 16
---

El «contexto» es **todo lo que OpenClaw envía al modelo para una ejecución**. Está limitado por la **ventana de contexto** del modelo (límite de tokens).

Modelo mental para principiantes:

- **Prompt del sistema** (creado por OpenClaw): reglas, herramientas, lista de Skills, hora/entorno de ejecución y archivos inyectados del espacio de trabajo.
- **Historial de conversación**: sus mensajes + los mensajes del asistente para esta sesión.
- **Llamadas/resultados de herramientas + archivos adjuntos**: salida de comandos, lecturas de archivos, imágenes/audio, etc.

El contexto _no es lo mismo_ que la «memoria»: la memoria puede almacenarse en disco y volver a cargarse posteriormente; el contexto es lo que se encuentra dentro de la ventana actual del modelo.

## Inicio rápido (inspeccionar el contexto)

- `/status` → vista rápida de «¿cuánto se ha llenado mi ventana?» + configuración de la sesión.
- `/context list` → qué se inyecta + tamaños aproximados (por archivo + totales).
- `/context detail` → desglose más detallado: tamaños por archivo, por esquema de herramienta, por entrada de Skill, tamaño del prompt del sistema y recuentos de mensajes de la transcripción que pueden compactarse.
- `/context map` → imagen de mapa de árbol al estilo de WinDirStat con los elementos registrados que contribuyen al contexto de la sesión actual.
- `/usage tokens` → añade un pie de uso por respuesta a las respuestas normales.
- `/compact` → resume el historial antiguo en una entrada compacta para liberar espacio en la ventana.

Consulte también: [Comandos de barra diagonal](/es/tools/slash-commands), [Uso y costes de tokens](/es/reference/token-use), [Compaction](/es/concepts/compaction).

## Ejemplo de salida

Los valores varían según el modelo, el proveedor, la política de herramientas y el contenido del espacio de trabajo.

### `/context list`

```text
🧠 Desglose del contexto
Espacio de trabajo: <workspaceDir>
Máximo de arranque/archivo: 12,000 caracteres
Entorno aislado: mode=non-main sandboxed=false
Prompt del sistema (ejecución): 38,412 caracteres (~9,603 tok) (Contexto del proyecto: 23,901 caracteres (~5,976 tok))

Archivos inyectados del espacio de trabajo:
- AGENTS.md: CORRECTO | sin procesar: 1,742 caracteres (~436 tok) | inyectado: 1,742 caracteres (~436 tok)
- SOUL.md: CORRECTO | sin procesar: 912 caracteres (~228 tok) | inyectado: 912 caracteres (~228 tok)
- TOOLS.md: TRUNCADO | sin procesar: 54,210 caracteres (~13,553 tok) | inyectado: 20,962 caracteres (~5,241 tok)
- IDENTITY.md: CORRECTO | sin procesar: 211 caracteres (~53 tok) | inyectado: 211 caracteres (~53 tok)
- USER.md: CORRECTO | sin procesar: 388 caracteres (~97 tok) | inyectado: 388 caracteres (~97 tok)
- HEARTBEAT.md: FALTA | sin procesar: 0 | inyectado: 0
- BOOTSTRAP.md: CORRECTO | sin procesar: 0 caracteres (~0 tok) | inyectado: 0 caracteres (~0 tok)

Lista de Skills (texto del prompt del sistema): 2,184 caracteres (~546 tok) (12 Skills)
Herramientas: read, edit, write, exec, process, browser, message, sessions_send, …
Lista de herramientas (texto del prompt del sistema): 1,032 caracteres (~258 tok)
Esquemas de herramientas (JSON): 31,988 caracteres (~7,997 tok) (cuentan para el contexto; no se muestran como texto)
Herramientas: (las mismas que arriba)

Tokens de la sesión (en caché): 14,250 en total / ctx=32,000
```

### `/context detail`

```text
🧠 Desglose del contexto (detallado)
…
Skills principales (tamaño de la entrada del prompt):
- frontend-design: 412 caracteres (~103 tok)
- oracle: 401 caracteres (~101 tok)
… (+10 Skills más)

Herramientas principales (tamaño del esquema):
- browser: 9,812 caracteres (~2,453 tok)
- exec: 6,240 caracteres (~1,560 tok)
… (+N herramientas más)
```

### `/context map`

Envía una imagen generada a partir del último informe de ejecución almacenado en caché y la transcripción de la sesión. Antes de que un mensaje normal haya producido un informe de ejecución en la sesión, `/context map` devuelve un mensaje de indisponibilidad en lugar de representar una estimación. El área de los rectángulos es proporcional a los caracteres registrados del prompt:

- transcripción de la conversación (mensajes del usuario, respuestas del asistente, resultados de herramientas y resúmenes de Compaction), además del contexto del entorno de ejecución por turno y las adiciones al prompt realizadas por enlaces que solo llegan al modelo
- archivos inyectados del espacio de trabajo
- texto base del prompt del sistema
- entradas del prompt de Skills
- esquemas JSON de herramientas

El grupo de conversación crece con la sesión, por lo que el mapa cambia de un turno a otro; después de la Compaction, se contrae en un mosaico de resúmenes.

`/context list`, `/context detail` y `/context json` aún pueden inspeccionar una estimación bajo demanda cuando no hay ningún informe de ejecución almacenado en caché.

## Qué cuenta para la ventana de contexto

Todo lo que recibe el modelo cuenta, incluido lo siguiente:

- Prompt del sistema (todas las secciones).
- Historial de conversación.
- Llamadas a herramientas + resultados de herramientas.
- Archivos adjuntos/transcripciones (imágenes/audio/archivos).
- Resúmenes de Compaction y artefactos de poda.
- «Envoltorios» del proveedor o encabezados ocultos (no visibles, pero también cuentan).

## Cómo crea OpenClaw el prompt del sistema

El prompt del sistema es **propiedad de OpenClaw** y se vuelve a crear en cada ejecución. Incluye:

- Lista de herramientas + descripciones breves.
- Lista de Skills (solo metadatos; consulte más adelante).
- Ubicación del espacio de trabajo.
- Hora (UTC + hora del usuario convertida, si está configurada).
- Metadatos del entorno de ejecución (host/SO/modelo/razonamiento).
- Archivos de arranque inyectados del espacio de trabajo bajo **Contexto del proyecto**.

Desglose completo: [Prompt del sistema](/es/concepts/system-prompt).

## Archivos inyectados del espacio de trabajo (Contexto del proyecto)

De forma predeterminada, OpenClaw inyecta un conjunto fijo de archivos del espacio de trabajo (si existen):

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo en la primera ejecución)

Los archivos grandes se truncan individualmente mediante `agents.defaults.bootstrapMaxChars` (valor predeterminado: `20000` caracteres). OpenClaw también aplica un límite total de inyección de arranque entre todos los archivos mediante `agents.defaults.bootstrapTotalMaxChars` (valor predeterminado: `60000` caracteres). `/context` muestra los tamaños **sin procesar frente a inyectados** y si se produjo un truncamiento.

Cuando se produce un truncamiento, el entorno de ejecución puede inyectar un bloque de advertencia en el prompt bajo Contexto del proyecto. Configure esta opción con `agents.defaults.bootstrapPromptTruncationWarning` (`off`, `once`, `always`; valor predeterminado: `always`).

## Skills: inyectadas frente a cargadas bajo demanda

El prompt del sistema incluye una **lista de Skills** compacta (nombre + descripción + ubicación). Esta lista tiene una sobrecarga real.

Las instrucciones de una Skill _no_ se incluyen de forma predeterminada. Se espera que el modelo `read` el archivo `SKILL.md` de la Skill **solo cuando sea necesario**.

## Herramientas: existen dos costes

Las herramientas afectan al contexto de dos maneras:

1. **Texto de la lista de herramientas** en el prompt del sistema (lo que se muestra como «Herramientas»).
2. **Esquemas de herramientas** (JSON). Se envían al modelo para que pueda llamar a las herramientas. Cuentan para el contexto aunque no se muestren como texto sin formato.

`/context detail` desglosa los esquemas de herramientas más grandes para mostrar cuáles predominan.

## Comandos, directivas y «atajos insertados»

El Gateway gestiona los comandos de barra diagonal. Existen varios comportamientos diferentes:

- **Comandos independientes**: un mensaje que solo contiene `/...` se ejecuta como comando.
- **Directivas**: `/think`, `/fast`, `/verbose`, `/trace`, `/reasoning`, `/elevated`, `/exec`, `/model` y `/queue` se eliminan antes de que el modelo vea el mensaje.
  - Los mensajes que solo contienen directivas conservan la configuración de la sesión.
  - Las directivas insertadas en un mensaje normal actúan como indicaciones para ese mensaje.
- **Atajos insertados** (solo para remitentes incluidos en la lista de permitidos): determinados tokens `/...` incluidos en un mensaje normal pueden ejecutarse inmediatamente (ejemplo: «hola /status») y se eliminan antes de que el modelo vea el texto restante.

Detalles: [Comandos de barra diagonal](/es/tools/slash-commands).

## Sesiones, Compaction y poda (qué se conserva)

Lo que se conserva entre mensajes depende del mecanismo:

- **Historial normal**: se conserva en la transcripción de la sesión hasta que la política lo compacta o poda.
- **Compaction**: conserva un resumen en la transcripción y mantiene intactos los mensajes recientes.
- **Poda**: elimina los resultados antiguos de herramientas del prompt _en memoria_ para liberar espacio en la ventana de contexto, pero no vuelve a escribir la transcripción de la sesión; el historial completo aún puede inspeccionarse en el disco.

Documentación: [Sesión](/es/concepts/session), [Compaction](/es/concepts/compaction), [Poda de sesiones](/es/concepts/session-pruning).

De forma predeterminada, OpenClaw utiliza el motor de contexto integrado `legacy` para el ensamblado y la
Compaction. Si se instala un Plugin que proporciona `kind: "context-engine"` y
se selecciona mediante `plugins.slots.contextEngine`, OpenClaw delega en ese motor el ensamblado del
contexto, `/compact` y los enlaces relacionados con el ciclo de vida del contexto de los subagentes.
`ownsCompaction: false` no recurre automáticamente al motor
heredado; el motor activo debe seguir implementando `compact()` correctamente. Consulte
[Motor de contexto](/es/concepts/context-engine) para conocer la interfaz
conectable completa, los enlaces del ciclo de vida y la configuración.

## Qué informa realmente `/context`

`/context` da preferencia al último informe del prompt del sistema **creado durante la ejecución** cuando está disponible:

- `System prompt (run)` = capturado durante la última ejecución integrada (con capacidad para usar herramientas) y conservado en el almacén de sesiones.
- `System prompt (estimate)` = calculado al vuelo cuando no existe un informe de ejecución (o cuando se ejecuta mediante un backend de CLI que no genera el informe).

En ambos casos, informa de los tamaños y de los principales elementos contribuyentes; **no** vuelca el prompt del sistema completo ni los esquemas de herramientas. En el modo detallado, también compara la transcripción de la sesión con el mismo predicado de mensajes de conversación reales que utiliza la Compaction, por lo que resulta más fácil distinguir un uso elevado del prompt o de la caché del historial de conversación que puede compactarse.

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Motor de contexto" href="/es/concepts/context-engine" icon="puzzle-piece">
    Inyección de contexto personalizada mediante Plugins.
  </Card>
  <Card title="Compaction" href="/es/concepts/compaction" icon="compress">
    Resumen de conversaciones largas para mantenerlas dentro de la ventana del modelo.
  </Card>
  <Card title="Prompt del sistema" href="/es/concepts/system-prompt" icon="message-lines">
    Cómo se crea el prompt del sistema y qué inyecta en cada turno.
  </Card>
  <Card title="Bucle del agente" href="/es/concepts/agent-loop" icon="arrows-rotate">
    El ciclo completo de ejecución del agente, desde el mensaje entrante hasta la respuesta final.
  </Card>
</CardGroup>
