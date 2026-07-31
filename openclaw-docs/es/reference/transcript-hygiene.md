---
read_when:
    - Está depurando rechazos de solicitudes del proveedor relacionados con la estructura de la transcripción
    - Está cambiando la lógica de saneamiento de transcripciones o de reparación de llamadas a herramientas
    - Está investigando discrepancias en los identificadores de llamadas a herramientas entre proveedores
summary: 'Referencia: reglas de saneamiento y reparación de transcripciones específicas del proveedor'
title: Higiene de las transcripciones
x-i18n:
    generated_at: "2026-07-26T04:51:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33d978772062cb2a81eb358bb5c62bd1261b433ffdc8acdbaa6679b121fbbf62
    source_path: reference/transcript-hygiene.md
    workflow: 16
---

OpenClaw aplica **correcciones específicas del proveedor** a las transcripciones antes de una ejecución
(al crear el contexto del modelo). La mayoría son ajustes **en memoria** que se utilizan para
cumplir los requisitos estrictos del proveedor. Una fase independiente de reparación del archivo de sesión también puede
reescribir el JSONL almacenado antes de cargar la sesión, pero solo para
líneas mal formadas o turnos persistentes que no constituyan registros duraderos válidos.
Las respuestas entregadas del asistente se conservan en el disco; la eliminación del
prellenado del asistente específica del proveedor solo se realiza al crear las cargas útiles
salientes.

Cuando se realiza una reparación, el archivo original se escribe en un archivo hermano transitorio
`*.bak-<pid>-<ts>` antes del reemplazo atómico y, a continuación, se elimina cuando el
reemplazo se completa correctamente. La copia de seguridad solo se conserva si falla la propia limpieza, en
cuyo caso se devuelve la ruta.

El alcance incluye:

- El contexto del prompt exclusivo del entorno de ejecución se mantiene fuera de los turnos de la transcripción visibles para el usuario
- Saneamiento del id de llamada a herramientas
- Validación de la entrada de llamadas a herramientas
- Reparación del emparejamiento de resultados de herramientas
- Validación / ordenación de turnos
- Limpieza de firmas de pensamientos
- Limpieza de firmas de razonamiento
- Saneamiento de cargas útiles de imágenes
- Limpieza de bloques de texto en blanco antes de la reproducción del proveedor
- Limpieza de turnos incompletos por longitud que solo contienen razonamiento antes de la reproducción del proveedor
- Etiquetado de procedencia de la entrada del usuario (para prompts enrutados entre sesiones)
- Reparación de turnos de error vacíos del asistente para la reproducción de Bedrock Converse

Si se necesitan detalles sobre el almacenamiento de transcripciones, consulte
[Análisis detallado de la gestión de sesiones](/es/reference/session-management-compaction).

---

## Regla global: el contexto del entorno de ejecución no es la transcripción del usuario

El contexto del entorno de ejecución o del sistema puede añadirse al prompt del modelo para un turno, pero
no es contenido creado por el usuario final. OpenClaw mantiene un cuerpo de prompt
independiente destinado a la transcripción para las respuestas del Gateway, los seguimientos en cola, ACP, CLI y las ejecuciones
integradas de OpenClaw. Los turnos visibles del usuario almacenados utilizan ese cuerpo de transcripción en lugar
del prompt enriquecido por el entorno de ejecución.

Para las sesiones heredadas que ya hayan conservado envoltorios del entorno de ejecución, las superficies del historial
del Gateway aplican una proyección de visualización antes de devolver los mensajes a los clientes
WebChat, TUI, REST o SSE.

---

## Dónde se ejecuta

Toda la higiene de transcripciones está centralizada en el ejecutor integrado:

- Selección de políticas: `src/agents/transcript-policy.ts`
  (`resolveTranscriptPolicy`, basada en `provider`, `modelApi` y `modelId`)
- Aplicación del saneamiento o la reparación: `sanitizeSessionHistory` en
  `src/agents/embedded-agent-runner/replay-history.ts`

Por separado de la higiene de transcripciones, los archivos de sesión se reparan (si es necesario)
antes de cargarse:

- `repairSessionFileIfNeeded` en `src/agents/session-file-repair.ts`
- Se llama desde `src/agents/embedded-agent-runner/run/attempt.ts` y
  `src/agents/embedded-agent-runner/compact.ts`

---

## Regla global: saneamiento de imágenes

Las cargas útiles de imágenes siempre se sanean para evitar que el proveedor las rechace debido a
los límites de tamaño (reducción de escala o recompresión de imágenes base64 sobredimensionadas). Esto también ayuda
a controlar la presión de tokens provocada por las imágenes en modelos con capacidades de visión: unas dimensiones
máximas menores reducen el uso de tokens, mientras que unas dimensiones mayores conservan los detalles.

Implementación:

- `sanitizeSessionMessagesImages` en
  `src/agents/embedded-agent-helpers/images.ts`
- `sanitizeContentBlocksImages` en `src/agents/tool-images.ts`
- El lado máximo de la imagen puede configurarse mediante `agents.defaults.imageMaxDimensionPx`
  (valor predeterminado: `1200`)
- Los bloques de texto en blanco se eliminan mientras esta fase recorre el contenido reproducido.
  Los turnos del asistente que quedan vacíos se descartan de la copia de reproducción; los turnos del usuario
  y de resultados de herramientas que quedan vacíos reciben un marcador de posición no vacío
  de contenido omitido.

---

## Regla global: llamadas a herramientas mal formadas

Los bloques de llamadas a herramientas del asistente a los que les falten tanto `input` como `arguments` se descartan
antes de crear el contexto del modelo. Esto evita que el proveedor rechace
llamadas a herramientas conservadas parcialmente (por ejemplo, después de un fallo por límite de velocidad).

Implementación:

- `sanitizeToolCallInputs` en `src/agents/session-transcript-repair.ts`
- Se aplica en `sanitizeSessionHistory`
  (`src/agents/embedded-agent-runner/replay-history.ts`)

---

## Regla global: emparejamiento de resultados de herramientas

Los resultados de herramientas se emparejan con las apariciones de llamadas a herramientas dentro de cada turno del asistente antes de
reescribir los id de llamada específicos del proveedor. Los id generados por el proveedor pueden repetirse en turnos
posteriores, por lo que un resultado adyacente a una llamada repetida permanece asociado a esa aparición. Un resultado
desplazado solo se mueve cuando exactamente una aparición sin resolver puede ser su propietaria; los resultados adicionales
ambiguos se descartan y las apariciones sin resultado reciben resultados de error sintéticos.

Implementación: `sanitizeToolUseResultPairing` en
`src/agents/session-transcript-repair.ts`

---

## Regla global: turnos incompletos o silenciosos que solo contienen razonamiento

Los turnos del asistente se omiten de la copia de reproducción en memoria cuando contienen
únicamente contenido de razonamiento o razonamiento censurado después de cualquiera de estos eventos:

- El límite de salida del proveedor finaliza el turno con un estado de razonamiento incompleto.
- La limpieza de respuestas silenciosas elimina el único texto visible `NO_REPLY` del turno.

La limpieza de respuestas silenciosas evita que el razonamiento oculto se combine con un turno posterior
de uso de herramientas del asistente cuando los proveedores estrictos reconstruyen la conversación.

Los turnos vacíos por longitud permanecen sin cambios, al igual que los turnos por longitud con texto visible,
llamadas a herramientas o bloques de contenido desconocido. Los turnos de respuesta silenciosa con llamadas a herramientas o
bloques de contenido desconocido también permanecen sin cambios. Las transcripciones almacenadas no se
reescriben.

Implementación: `normalizeAssistantReplayContent` en
`src/agents/embedded-agent-runner/replay-history.ts`

---

## Regla global: procedencia de entradas entre sesiones

Cuando un agente envía un prompt a otra sesión mediante `sessions_send`
(incluidos los pasos de respuesta o anuncio entre agentes), OpenClaw conserva el
turno de usuario creado con `message.provenance.kind = "inter_session"`.

OpenClaw también antepone un marcador `[Inter-session message] ... isUser=false` en el mismo turno
antes del texto del prompt enrutado para que la llamada activa al modelo pueda
distinguir la salida de una sesión ajena de las instrucciones externas del usuario final. Este
marcador incluye la sesión, el canal y la herramienta de origen cuando están disponibles. La
transcripción sigue utilizando `role: "user"` para la compatibilidad con el proveedor, pero tanto el
texto visible como los metadatos de procedencia marcan el turno como datos
entre sesiones.

Durante la reconstrucción del contexto, OpenClaw aplica el mismo marcador a los turnos de usuario
entre sesiones conservados anteriormente que solo tienen metadatos de procedencia.

---

## Matriz de proveedores (comportamiento actual)

**OpenAI / OpenAI Codex**

- Solo saneamiento de imágenes.
- Se descartan las firmas de razonamiento huérfanas (elementos de razonamiento independientes sin un
  bloque de contenido posterior) en las transcripciones de OpenAI Responses/Codex, y se descarta
  el razonamiento reproducible de OpenAI después de cambiar la ruta del modelo.
- Se conservan las cargas útiles reproducibles de los elementos de razonamiento de OpenAI Responses, incluidos
  los elementos cifrados con resumen vacío, para que la reproducción manual o mediante WebSocket mantenga el estado
  `rs_*` requerido emparejado con los elementos de salida del asistente.
- Responses de ChatGPT Codex nativo mantiene la paridad con el protocolo de Codex mediante la reproducción
  de las cargas útiles anteriores de razonamiento, mensajes y funciones de Responses sin los id de elementos
  anteriores, a la vez que conserva el `prompt_cache_key` de la sesión.
- La reproducción de la familia OpenAI Responses conserva los pares canónicos de razonamiento
  `call_*|fc_*` del mismo modelo, pero normaliza de forma determinista los id de elementos
  `call_id` o de llamadas a funciones que estén mal formados o sean demasiado largos antes de convertir la carga útil de pi-ai.
- La reparación del emparejamiento de resultados de herramientas puede mover salidas reales coincidentes y sintetizar
  salidas `aborted` al estilo de Codex para llamadas a herramientas sin resultado.
- No se validan ni reordenan los turnos; no se eliminan las firmas de pensamientos.

**Chat Completions compatibles con OpenAI**

- Los bloques históricos de pensamiento o razonamiento del asistente se eliminan antes de la reproducción
  para que los servidores locales y de tipo proxy compatibles con OpenAI no reciban
  campos de razonamiento de turnos anteriores como `reasoning` o `reasoning_content`.
- Las continuaciones de llamadas a herramientas del mismo turno actual mantienen el bloque de razonamiento
  del asistente asociado a la llamada a herramientas hasta que se haya reproducido el resultado de la herramienta.
- Las entradas de modelos personalizados o autoalojados con `reasoning: true` conservan los metadatos
  de razonamiento reproducidos.
- Las excepciones pertenecientes al proveedor pueden excluirse cuando su protocolo requiere
  metadatos de razonamiento reproducidos.

**Google (IA generativa / Gemini CLI / Antigravity)**

- Saneamiento del id de llamada a herramientas: estrictamente alfanumérico.
- Reparación del emparejamiento de resultados de herramientas y resultados de herramientas sintéticos.
- Validación de turnos (alternancia de turnos al estilo de Gemini).
- Corrección del orden de turnos de Google (se antepone un pequeño mensaje de inicio del usuario si el historial
  comienza con el asistente).
- Claude de Antigravity: se normalizan las firmas de razonamiento y se descartan los bloques de razonamiento
  sin firma.

**Anthropic / Minimax (compatible con Anthropic)**

- Reparación del emparejamiento de resultados de herramientas y resultados de herramientas sintéticos.
- Validación de turnos (se combinan turnos consecutivos del usuario para cumplir la
  alternancia estricta).
- Los turnos finales de prellenado del asistente se eliminan de las cargas útiles salientes de Anthropic
  Messages cuando el razonamiento está habilitado, incluidas las rutas de Cloudflare AI
  Gateway.
- Las firmas de razonamiento del asistente anteriores a la Compaction se eliminan antes de la reproducción
  del proveedor cuando se ha compactado una sesión. Las firmas de razonamiento están
  vinculadas criptográficamente al prefijo de la conversación en el momento de generarse;
  después de la Compaction, el prefijo cambia (el contenido resumido sustituye al
  original), por lo que reproducir las firmas originales hace que Anthropic
  rechace la solicitud con "Invalid signature in thinking block". El
  texto del razonamiento se conserva como un bloque sin firma y después se procesa mediante la
  regla siguiente.
- Los bloques de razonamiento cuyas firmas de reproducción falten, estén vacías o estén en blanco se
  eliminan antes de la conversión del proveedor. Si esto vacía un turno del asistente,
  OpenClaw conserva la estructura del turno con texto no vacío que indica que se ha omitido el razonamiento.
- Los turnos antiguos del asistente que solo contienen razonamiento y deben eliminarse se sustituyen
  por texto no vacío que indica que se ha omitido el razonamiento, para que los adaptadores del proveedor no descarten
  el turno reproducido.

**Amazon Bedrock (API Converse)**

- Los turnos vacíos del asistente con errores de transmisión se reparan con un bloque de texto alternativo
  no vacío antes de la reproducción. Bedrock Converse rechaza los mensajes del asistente
  con `content: []`, por lo que los turnos persistentes del asistente con `stopReason:
"error"` y contenido vacío también se reparan en el disco antes de cargarlos.
- Los turnos del asistente con errores de transmisión que solo contienen bloques de texto en blanco se descartan de
  la copia de reproducción en memoria en lugar de reproducir un bloque en blanco no válido.
- Las firmas de razonamiento del asistente anteriores a la Compaction se eliminan antes de la reproducción de Converse
  cuando se ha compactado una sesión, por el mismo motivo que en Anthropic
  anteriormente.
- Los bloques de razonamiento de Claude cuyas firmas de reproducción falten, estén vacías o estén en blanco
  se eliminan antes de la reproducción de Converse. Si esto vacía un turno del asistente,
  OpenClaw conserva la estructura del turno con texto no vacío que indica que se ha omitido el razonamiento.
- Los turnos antiguos del asistente que solo contienen razonamiento y deben eliminarse se sustituyen
  por texto no vacío que indica que se ha omitido el razonamiento, para que la reproducción de Converse conserve
  la estructura estricta de los turnos.
- La reproducción filtra los turnos del asistente reflejados para la entrega de OpenClaw e inyectados por
  el Gateway.
- El saneamiento de imágenes se aplica mediante la regla global.

**Mistral (incluida la detección basada en el id del modelo)**

- Saneamiento del id de llamada a herramientas: strict9 (alfanumérico, longitud 9).

**Gemini de OpenRouter**

- Limpieza de firmas de pensamientos: se eliminan los valores `thought_signature` que no sean base64
  (se conservan los que sean base64).

**Anthropic de OpenRouter**

- Los turnos finales de prellenado del asistente se eliminan de las cargas útiles verificadas de modelos Anthropic
  de OpenRouter compatibles con OpenAI cuando el razonamiento está habilitado,
  de acuerdo con el comportamiento de reproducción de Anthropic directo y Anthropic mediante Cloudflare.

**Todo lo demás**

- Solo saneamiento de imágenes.

---

## Comportamiento histórico (anterior a 2026.1.22)

Antes de la versión 2026.1.22, OpenClaw aplicaba varias capas de higiene de
transcripciones:

- Una **extensión transcript-sanitize** se ejecutaba en cada creación de contexto y podía:
  - Reparar el emparejamiento entre el uso de herramientas y sus resultados.
  - Sanitizar los identificadores de llamadas a herramientas (incluido un modo no estricto que conservaba
    `_`/`-`).
- El ejecutor también realizaba una sanitización específica del proveedor, lo que
  duplicaba el trabajo.
- Se producían mutaciones adicionales fuera de la política del proveedor, como
  eliminar las etiquetas `<final>` del texto del asistente antes de la persistencia, descartar
  turnos de error vacíos del asistente y recortar el contenido del asistente después de las llamadas a
  herramientas.

Esta complejidad causaba regresiones entre proveedores (en particular, en el
emparejamiento de `openai-responses` y `call_id|fc_id`). La limpieza de 2026.1.22 eliminó
la extensión, centralizó la lógica en el ejecutor e hizo que OpenAI quedara **sin modificaciones**
más allá de la sanitización de imágenes.

## Contenido relacionado

- [Gestión de sesiones](/es/concepts/session)
- [Poda de sesiones](/es/concepts/session-pruning)
