---
read_when:
    - Diagnóstico del comportamiento de rotación de perfiles de autenticación, períodos de espera o conmutación por error del modelo
    - Actualización de las reglas de conmutación por error para perfiles de autenticación o modelos
    - Comprender cómo interactúan las sustituciones del modelo de sesión con los reintentos de respaldo
sidebarTitle: Model failover
summary: Cómo OpenClaw rota los perfiles de autenticación y recurre a modelos alternativos
title: Conmutación por error de modelos
x-i18n:
    generated_at: "2026-07-26T04:35:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dfedbc85038eebb5be056a7b3ffa3275b4329a0b0d791e1a2b4701cbaa4b595
    source_path: concepts/model-failover.md
    workflow: 16
---

OpenClaw gestiona los fallos en dos etapas:

1. **Rotación de perfiles de autenticación** dentro del proveedor actual.
2. **Cambio al modelo de respaldo** al siguiente modelo en `agents.defaults.model.fallbacks`.

## Flujo de ejecución

<Steps>
  <Step title="Resolver el estado de la sesión">
    Resuelve el modelo de la sesión activa y la preferencia de perfil de autenticación.
  </Step>
  <Step title="Crear la cadena de candidatos">
    Crea la cadena de modelos candidatos a partir de la selección de modelo actual y la política de respaldo para el origen de esa selección. Los valores predeterminados configurados, los modelos principales de tareas cron y los modelos de respaldo seleccionados automáticamente pueden usar los respaldos configurados; las selecciones explícitas de la sesión del usuario son estrictas.
  </Step>
  <Step title="Probar el proveedor actual">
    Prueba el proveedor actual con las reglas de rotación y enfriamiento de perfiles de autenticación.
  </Step>
  <Step title="Avanzar ante errores que justifican la conmutación por error">
    Si se agota ese proveedor con un error que justifica la conmutación por error, pasa al siguiente modelo candidato.
  </Step>
  <Step title="Usar el respaldo para el turno actual">
    Ejecuta el candidato de respaldo satisfactorio sin cambiar el proveedor ni el modelo seleccionados para la sesión.
  </Step>
  <Step title="Reintentar cuando solo se agoten los recursos por sobrecarga">
    Si todos los candidatos fallan únicamente porque los proveedores están sobrecargados, reintenta la cadena local completa del turno hasta 10 veces con retroceso exponencial, siempre que no se haya iniciado ninguna ejecución de herramientas ni salida del asistente. Después de 30 segundos, envía un aviso de estado para no dejar al usuario esperando en silencio.
  </Step>
  <Step title="Generar FallbackSummaryError si se agotan las opciones">
    Si todos los candidatos fallan, genera un `FallbackSummaryError` con los detalles de cada intento y la expiración de enfriamiento más próxima, cuando se conozca.
  </Step>
</Steps>

La ejecución del respaldo es local al turno. El ejecutor de respuestas conserva únicamente el estado de los avisos de respaldo para que `/status` y los avisos de transición puedan distinguir el modelo seleccionado del modelo que respondió; no conserva el respaldo como selección de modelo del siguiente turno.

## Política del origen de selección

El origen de selección determina si se permite la cadena de respaldo:

- **Valor predeterminado configurado**: `agents.defaults.model.primary` usa `agents.defaults.model.fallbacks`.
- **Modelo principal del agente**: `agents.entries.*.model` es estricto, salvo que el objeto de modelo de ese agente incluya su propio `fallbacks`. Usa `fallbacks: []` para hacer explícito el comportamiento estricto o una lista no vacía para habilitar el cambio de modelo de respaldo para ese agente.
- **Respaldo durante la ejecución**: el candidato de respaldo se aplica únicamente al turno actual. El siguiente turno comienza de nuevo con el modelo principal seleccionado. OpenClaw sigue reconociendo las entradas `modelOverrideSource: "auto"` almacenadas anteriormente, comprueba su origen configurado cada 5 minutos y las elimina cuando el origen se recupera. `/new`, `/reset` y `sessions.reset` también eliminan esas entradas.
- **Anulación de la sesión por el usuario**: `/model`, el selector de modelos, `session_status(model=...)` y `sessions.patch` escriben `modelOverrideSource: "user"`. Esta es una selección exacta para la sesión. Si el proveedor o modelo seleccionado falla antes de producir una respuesta, OpenClaw informa del fallo en lugar de responder desde un respaldo configurado no relacionado.
- **Anulación de sesión heredada**: las entradas de sesión antiguas pueden tener `modelOverride` sin `modelOverrideSource`. OpenClaw las trata como anulaciones del usuario para que una selección antigua explícita no se convierta silenciosamente en un comportamiento de respaldo.
- **Modelo de la carga útil de Cron**: un `payload.model` / `--model` de una tarea cron es el modelo principal de la tarea, no una anulación de sesión del usuario. Usa los respaldos configurados, salvo que la tarea proporcione `payload.fallbacks`; `payload.fallbacks: []` hace que la ejecución cron sea estricta.

OpenClaw envía un aviso visible cuando un turno pasa a un respaldo y otro aviso cuando un turno posterior se completa correctamente con el modelo principal seleccionado. El estado conservado de los avisos evita que se repitan cuando varios turnos consecutivos usan el mismo par seleccionado/activo, mientras que la selección del modelo permanece sin cambios.

## Caché de omisión de fallos de autenticación

De forma predeterminada, cada turno nuevo mantiene el comportamiento existente de reintento de respaldo: OpenClaw vuelve a intentar cada candidato de respaldo configurado, incluidos los candidatos no principales que han fallado recientemente con `auth` o `auth_permanent`.

Para suprimir los fallos de autenticación repetidos, habilita:

```bash
OPENCLAW_FALLBACK_SKIP_TTL_MS=60000
```

Cuando está habilitado, OpenClaw registra en memoria un marcador de omisión limitado a la sesión para un candidato de respaldo no principal después de un fallo de la clase de autenticación, identificado por el ID de sesión, el proveedor y el modelo. Los candidatos principales nunca se omiten, por lo que una selección explícita de modelo por parte del usuario sigue mostrando el error de autenticación real. La caché es local al proceso y se borra al reiniciar el Gateway.

El valor es un TTL en milisegundos. `0` o la ausencia del valor deshabilitan la caché. Los valores positivos se limitan a un intervalo de entre 1 segundo y 10 minutos.

## Avisos de respaldo visibles para el usuario

Cuando una sesión pasa a un respaldo seleccionado automáticamente, OpenClaw envía un aviso de estado en la misma superficie de respuesta:

```text
↪️ Modelo de respaldo: <fallback> (seleccionado <primary>; <reason>)
```

Cuando una comprobación posterior se completa correctamente y la sesión vuelve al modelo principal seleccionado, OpenClaw envía:

```text
↪️ Respaldo de modelo desactivado: <primary> (era <fallback>)
```

Estos avisos son mensajes operativos, no contenido del asistente. Se entregan una vez por cada cambio de estado, incluidos, cuando sea posible, los turnos que solo producen efectos secundarios, pero las transiciones repetidas al respaldo local del turno no hacen que vuelvan a enviarse. La entrega omite la supresión normal de respuestas al origen, no consume el primer espacio de respuesta del asistente en los canales con hilos y se excluye de la conversión de texto a voz y de la extracción de compromisos.

## Almacenamiento de autenticación (claves + OAuth)

OpenClaw usa **perfiles de autenticación** tanto para claves de API como para tokens de OAuth.

- Los secretos y el estado de enrutamiento de autenticación durante la ejecución residen en `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`.
- Los valores de configuración `auth.profiles` / `auth.order` son **solo metadatos y enrutamiento** (sin secretos).
- Archivo OAuth heredado solo para importación: `~/.openclaw/credentials/oauth.json` (se importa en el almacén de autenticación por agente al usarse por primera vez).
- Los archivos heredados `auth-profiles.json`, `auth-state.json` y los archivos `auth.json` por agente se importan mediante `openclaw doctor --fix`.

Más información: [OAuth](/es/concepts/oauth)

Tipos de credenciales:

- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }` (+ `projectId`/`enterpriseUrl` para algunos proveedores)
- `type: "token"` → token estático de tipo portador, con caducidad opcional; OpenClaw no lo renueva (se usa para `aws-sdk` y otros modos de autenticación mediante cadenas de credenciales)

## ID de perfiles

Los inicios de sesión con OAuth crean perfiles distintos para que puedan coexistir varias cuentas.

- Valor predeterminado: `provider:default` cuando no hay ninguna dirección de correo electrónico disponible.
- OAuth con correo electrónico: `provider:<email>` (por ejemplo, `google-antigravity:user@gmail.com`).

Los perfiles residen en el almacén de perfiles de autenticación `openclaw-agent.sqlite` por agente.

## Orden de rotación

Cuando un proveedor tiene varios perfiles, OpenClaw elige un orden de la siguiente manera:

<Steps>
  <Step title="Configuración explícita">
    `auth.order[provider]` (si está definido).
  </Step>
  <Step title="Perfiles configurados">
    `auth.profiles` filtrados por proveedor.
  </Step>
  <Step title="Perfiles almacenados">
    Entradas de perfiles de autenticación de SQLite por agente para el proveedor.
  </Step>
</Steps>

Si no se configura un orden explícito, OpenClaw usa un orden por turnos:

- **Clave principal:** tipo de perfil (**OAuth, después token estático y, por último, clave de API**).
- **Clave secundaria para OAuth:** perfiles con un token de acceso utilizable actualmente antes que
  los perfiles cuyo token de acceso ha caducado. Los perfiles OAuth caducados siguen siendo aptos para que
  el entorno de ejecución pueda renovarlos cuando no haya ningún perfil equivalente utilizable.
- **Clave siguiente:** `usageStats.lastUsed` (primero el más antiguo, dentro de cada nivel de tipo/estado).
- Los **perfiles en enfriamiento o deshabilitados** se trasladan al final, ordenados por la fecha de expiración más próxima.

### Afinidad de sesión (favorable para la caché)

OpenClaw **fija el perfil de autenticación elegido para cada sesión** con el fin de mantener activas las cachés del proveedor. **No** lo rota en cada solicitud. El perfil fijado se reutiliza hasta que:

- se restablece la sesión (`/new` / `/reset`)
- finaliza una Compaction (aumenta el contador de Compaction)
- el perfil entra en enfriamiento o se deshabilita

La selección manual mediante `/model …@<profileId>` establece una **anulación del usuario** para esa sesión y no se rota automáticamente hasta que comienza una sesión nueva.

<Note>
Los perfiles fijados automáticamente (seleccionados por el enrutador de sesiones) se tratan como una **preferencia**: se prueban primero, pero OpenClaw puede rotar a otro perfil ante límites de frecuencia o tiempos de espera. Cuando el perfil original vuelve a estar disponible, las ejecuciones nuevas pueden volver a preferirlo sin cambiar el modelo ni el entorno de ejecución seleccionados. Los perfiles fijados por el usuario permanecen vinculados a ese perfil; si falla y hay respaldos de modelos configurados, OpenClaw pasa al siguiente modelo en lugar de cambiar de perfil.
</Note>

### Suscripción a OpenAI Codex con respaldo mediante clave de API

Para los modelos de agente de OpenAI, la autenticación y el entorno de ejecución son independientes. `openai/gpt-*` permanece en el entorno de Codex, mientras que la autenticación puede rotar entre un perfil de suscripción a Codex y un respaldo mediante clave de API de OpenAI.

Usa `auth.order.openai` para el orden visible para el usuario:

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Usa `openai:*` tanto para los perfiles OAuth de ChatGPT/Codex como para los perfiles de clave de API de OpenAI. Cuando la suscripción alcanza un límite de uso de Codex, OpenClaw registra la hora exacta de restablecimiento si Codex la proporciona, prueba el siguiente perfil de autenticación ordenado y mantiene la ejecución dentro del entorno de Codex. Una vez transcurrida la hora de restablecimiento, el perfil de suscripción vuelve a ser apto y la siguiente selección automática puede volver a él.

Usa un perfil fijado por el usuario solo cuando se quiera forzar una cuenta o clave para esa sesión. Los perfiles fijados por el usuario son intencionadamente estrictos y no pasan silenciosamente a otro perfil.

## Enfriamientos

Cuando un perfil falla debido a errores de autenticación o de límite de frecuencia (o a un tiempo de espera que parece deberse a un límite de frecuencia), OpenClaw lo marca en enfriamiento y pasa al siguiente perfil.

<AccordionGroup>
  <Accordion title="Qué se incluye en la categoría de límite de frecuencia o tiempo de espera">
    Esa categoría de límite de frecuencia es más amplia que el simple `429`: también incluye mensajes del proveedor como `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`, `throttled`, `resource exhausted` y límites periódicos de ventanas de uso como `weekly limit reached` o `monthly limit exhausted`.

    Los errores de formato o de solicitud no válida suelen ser terminales porque volver a intentar la misma carga útil fallaría de la misma manera, por lo que OpenClaw los muestra en lugar de rotar los perfiles de autenticación. Las rutas conocidas de reparación y reintento pueden habilitarse explícitamente: por ejemplo, los fallos de validación del ID de llamada a herramientas de Cloud Code Assist se depuran y se reintentan una vez mediante la política `allowFormatRetry`.

    Los motivos de detención o finalización de tipo **finalizado por el proveedor** compatibles con OpenAI, como `Unhandled stop reason: error`, `stop reason: error`, `reason: error` y `Provider finish_reason: error`, se clasifican como **`server_error`** (estado similar a HTTP 500), no como tiempo de espera. Siguen permitiendo la conmutación por error para la rotación de modelos o perfiles, pero los diagnósticos conservan el texto del motivo de finalización del proveedor en lugar de sustituir el texto mostrado al usuario por «Se agotó el tiempo de espera de la solicitud al LLM». Los motivos de finalización relacionados con el transporte, como `Provider finish_reason: abort`, `network_error` y `malformed_response`, permanecen en la categoría de tiempo de espera o conmutación por error (estado 408).

    El texto genérico del servidor también puede incluirse en esa categoría de tiempo de espera cuando el origen coincide con un patrón transitorio conocido. Por ejemplo, el mensaje simple del contenedor de flujos del entorno de ejecución del modelo `An unknown error occurred` se considera apto para la conmutación por error para todos los proveedores porque el entorno compartido de ejecución del modelo lo emite cuando los flujos del proveedor terminan con `stopReason: "aborted"` o `stopReason: "error"` sin detalles específicos. Las cargas útiles JSON `api_error` con texto transitorio del servidor, como `internal server error`, `unknown error, 520`, `upstream error` o `backend error`, también se consideran tiempos de espera aptos para la conmutación por error.

    El texto ascendente genérico específico de OpenRouter, como `Provider returned error` sin más, se trata como tiempo de espera agotado solo cuando el contexto del proveedor es realmente OpenRouter. El texto de respaldo interno genérico, como `LLM request failed with an unknown error.`, mantiene una clasificación conservadora y no activa la conmutación por error por sí solo.

  </Accordion>
  <Accordion title="Límites de retry-after del SDK">
    De lo contrario, algunos SDK de proveedores pueden permanecer en espera durante un intervalo `Retry-After` prolongado antes de devolver el control a OpenClaw. Para los SDK basados en Stainless, como los de Anthropic y OpenAI, OpenClaw limita de forma predeterminada a 60 segundos las esperas `retry-after-ms` / `retry-after` internas del SDK y presenta de inmediato las respuestas reintentables con esperas más largas para que pueda ejecutarse esta ruta de conmutación por error. Ajuste o desactive el límite con `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS`; consulte [Comportamiento de los reintentos](/es/concepts/retry).
  </Accordion>
  <Accordion title="Tiempos de espera por modelo">
    Los tiempos de espera por límite de frecuencia también pueden aplicarse a un modelo específico:

    - OpenClaw registra `cooldownModel` para los fallos por límite de frecuencia cuando se conoce el id. del modelo que falla.
    - Aún se puede probar otro modelo del mismo proveedor cuando el tiempo de espera se aplica a un modelo diferente.
    - Los periodos por facturación o desactivación siguen bloqueando todo el perfil en todos los modelos.

  </Accordion>
</AccordionGroup>

Los tiempos de espera normales (no debidos a facturación ni a fallos permanentes de autenticación) aumentan según el recuento de errores recientes del perfil:

- 1.er fallo: 30 segundos
- 2.º fallo: 1 minuto
- 3.er fallo y posteriores: 5 minutos (límite)

Los contadores se restablecen una vez transcurrido el periodo de fallos integrado del perfil.

El estado se almacena en el estado de autenticación SQLite por agente, en `usageStats`:

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## Desactivaciones por facturación

Los fallos de facturación o crédito (por ejemplo, "créditos insuficientes" / "saldo de crédito demasiado bajo") se consideran aptos para la conmutación por error, pero normalmente no son transitorios. En lugar de aplicar un breve tiempo de espera, OpenClaw marca el perfil como **desactivado** (con una espera exponencial más larga) y pasa al siguiente perfil o proveedor.

<Note>
No todas las respuestas con apariencia de fallo de facturación son `402`, ni todos los `402` HTTP terminan aquí. OpenClaw mantiene el texto explícito de facturación en la categoría de facturación aunque un proveedor devuelva `401` o `403`, pero los patrones de coincidencia específicos de un proveedor siguen limitados al proveedor al que pertenecen (por ejemplo, `403 Key limit exceeded` de OpenRouter).

Mientras tanto, los errores temporales de ventana de uso y de límite de gasto de organización o espacio de trabajo `402` se clasifican como `rate_limit` cuando el mensaje parece reintentable (por ejemplo, `weekly usage limit exhausted`, `daily limit reached, resets tomorrow` o `organization spending limit exceeded`). Estos permanecen en la ruta de tiempo de espera breve y conmutación por error, en lugar de seguir la ruta de desactivación prolongada por facturación.
</Note>

Los fallos permanentes de autenticación con alta certeza (claves revocadas o desactivadas y espacios de trabajo desactivados) siguen una categoría de desactivación similar, pero se recuperan mucho antes que los de facturación, ya que algunos proveedores muestran temporalmente cargas útiles con apariencia de fallo de autenticación durante los incidentes.

El estado se almacena en el estado de autenticación SQLite por agente:

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

Los errores de sobrecarga y de límite de frecuencia se gestionan de forma más agresiva que los tiempos de espera por facturación: de forma predeterminada, OpenClaw permite un reintento con otro perfil de autenticación del mismo proveedor y, a continuación, cambia al siguiente modelo de respaldo configurado sin esperar.

## Modelo de respaldo

Si fallan todos los perfiles de un proveedor, OpenClaw pasa al siguiente modelo de `agents.defaults.model.fallbacks`. Esto se aplica a los fallos de autenticación, los límites de frecuencia y los tiempos de espera agotados que hayan agotado la rotación de perfiles (los demás errores no hacen avanzar el respaldo). Los errores del proveedor que no muestran suficiente información se etiquetan de forma precisa en el estado del respaldo: `empty_response` significa que el proveedor no devolvió ningún mensaje o estado utilizable, `no_error_details` significa que el proveedor devolvió explícitamente `Unknown error (no error details in response)` y `unclassified` significa que OpenClaw conservó la vista previa sin procesar, pero aún no coincidió con ningún clasificador.

Las señales de proveedor ocupado, como `ModelNotReadyException`, se incluyen en la categoría de sobrecarga y siguen la misma política de una rotación y posterior respaldo que los límites de frecuencia (consulte la tabla de valores predeterminados anterior).

Si toda la cadena de candidatos se agota únicamente por fallos de sobrecarga, el ejecutor de respuestas reintenta la cadena hasta 10 veces en el mismo turno. El reintento del turno completo solo se permite antes de que comience la ejecución de herramientas o la salida del asistente, para evitar mutaciones o mensajes duplicados si se produce una sobrecarga después de un trabajo observable. La espera exponencial comienza en 2.5 segundos y se duplica hasta alcanzar un límite de 30 segundos. Cuando el turno lleva 30 segundos esperando, OpenClaw envía un único aviso de estado transitorio: `The AI service is temporarily overloaded. I’m still retrying; this may take a few minutes.` El reintento y cualquier candidato de respaldo que resulte válido permanecen limitados al turno; los errores transitorios normales del servidor mantienen su propia política independiente de un reintento.

Cuando una ejecución comienza desde el modelo principal predeterminado configurado, el modelo principal de una tarea Cron, el modelo principal de un agente con respaldos explícitos o una sustitución de respaldo seleccionada automáticamente, OpenClaw puede recorrer la cadena de respaldos configurada correspondiente. Los modelos principales de agentes sin respaldos explícitos y las selecciones explícitas del usuario (por ejemplo, `/model ollama/qwen3.5:27b`, el selector de modelos, `sessions.patch` o sustituciones puntuales del proveedor o modelo mediante la CLI) son estrictos: si no se puede acceder a ese proveedor o modelo, o falla antes de producir una respuesta, OpenClaw informa del fallo en lugar de responder mediante un respaldo no relacionado.

### Reglas de la cadena de candidatos

OpenClaw crea la lista de candidatos a partir de `provider/model` solicitado actualmente y los respaldos configurados.

<AccordionGroup>
  <Accordion title="Reglas">
    - El modelo solicitado siempre ocupa el primer lugar.
    - Los respaldos configurados explícitamente se deduplican, pero no se filtran mediante la lista de modelos permitidos. Se consideran una intención explícita del operador.
    - Si la ejecución actual ya utiliza un respaldo configurado de la misma familia de proveedores, OpenClaw continúa utilizando toda la cadena configurada.
    - Cuando no se proporciona una sustitución explícita de respaldos, los respaldos configurados se prueban antes que el modelo principal configurado, aunque el modelo solicitado utilice un proveedor diferente.
    - Cuando no se proporciona una sustitución explícita al ejecutor de respaldos, el modelo principal configurado se añade al final para que la cadena pueda volver al valor predeterminado normal una vez agotados los candidatos anteriores.
    - Cuando un llamador proporciona `fallbacksOverride`, el ejecutor utiliza exactamente el modelo solicitado y esa lista de sustitución. Una lista vacía desactiva el modelo de respaldo e impide que el modelo principal configurado se añada como destino de reintento oculto.

  </Accordion>
</AccordionGroup>

### Errores que hacen avanzar el respaldo

<Tabs>
  <Tab title="Continúa con">
    - fallos de autenticación
    - límites de frecuencia y agotamiento del tiempo de espera
    - errores de sobrecarga o proveedor ocupado
    - errores de conmutación por error con apariencia de tiempo de espera agotado
    - desactivaciones por facturación
    - `LiveSessionModelSwitchError`, que se normaliza en una ruta de conmutación por error para que un modelo persistido obsoleto no cree un bucle de reintentos externo
    - otros errores no reconocidos cuando aún quedan candidatos

  </Tab>
  <Tab title="No continúa con">
    - cancelaciones explícitas que no tienen apariencia de tiempo de espera agotado o conmutación por error
    - errores de desbordamiento de contexto que deben permanecer dentro de la lógica de Compaction y reintento (por ejemplo, `request_too_large`, `input token count exceeds the maximum number of input tokens`, `input exceeds the maximum number of tokens`, `input too long for the model` o `ollama error: context length exceeded`)
    - un error desconocido final cuando no quedan candidatos
    - rechazos de seguridad de Claude Fable 5; las solicitudes directas con clave de API los gestionan a nivel del proveedor mediante el respaldo de Anthropic del lado del servidor a `claude-opus-4-8` (consulte [Anthropic](/es/providers/anthropic#safety-refusal-fallback-claude-fable-5))

  </Tab>
</Tabs>

### Comportamiento al omitir el tiempo de espera frente a realizar una prueba

Cuando todos los perfiles de autenticación de un proveedor ya están en tiempo de espera, OpenClaw no omite automáticamente ese proveedor para siempre. Toma una decisión para cada candidato:

<AccordionGroup>
  <Accordion title="Decisiones por candidato">
    - Los fallos persistentes de autenticación hacen que se omita inmediatamente todo el proveedor.
    - Las desactivaciones por facturación normalmente hacen que se omita el proveedor, pero aún se puede probar el candidato principal con una limitación de frecuencia para permitir la recuperación sin reiniciar.
    - El candidato principal puede probarse cerca del vencimiento del tiempo de espera, con una limitación por proveedor.
    - Los respaldos alternativos del mismo proveedor pueden intentarse a pesar del tiempo de espera cuando el fallo parece transitorio (`rate_limit`, `overloaded` o desconocido). Esto resulta especialmente relevante cuando un límite de frecuencia se aplica a un modelo específico y otro modelo puede recuperarse de inmediato.
    - Las pruebas durante tiempos de espera transitorios se limitan a una por proveedor y ejecución del respaldo para evitar que un solo proveedor bloquee el respaldo entre proveedores.

  </Accordion>
</AccordionGroup>

## Sustituciones de sesión y cambio de modelo en directo

Los cambios del modelo de sesión forman parte del estado compartido. El ejecutor activo, el comando `/model`, las actualizaciones de Compaction o sesión y la reconciliación de la sesión en directo leen o escriben partes de la misma entrada de sesión. La ejecución del respaldo no escribe campos de selección del modelo, por lo que no puede sustituir una selección manual más reciente mientras realiza reintentos.

El cambio de modelo en directo sigue estas reglas:

- Solo los cambios explícitos de modelo iniciados por el usuario marcan un cambio en directo pendiente. Esto incluye `/model`, `session_status(model=...)` y `sessions.patch`.
- Los cambios de modelo iniciados por el sistema, como la rotación del respaldo, las sustituciones de Heartbeat o la Compaction, nunca marcan por sí solos un cambio en directo pendiente.
- Las sustituciones de modelo iniciadas por el usuario se tratan como selecciones exactas para la política de respaldo, por lo que un proveedor seleccionado inaccesible se muestra como un fallo en lugar de quedar oculto por `agents.defaults.model.fallbacks`.
- Los candidatos de respaldo en tiempo de ejecución permanecen limitados al turno. El siguiente turno comienza desde el modelo seleccionado actualmente, incluida una selección manual realizada durante la ejecución anterior.
- Las sustituciones de respaldo automáticas almacenadas previamente siguen siendo compatibles: OpenClaw prueba periódicamente su origen configurado y elimina la sustitución cuando este se recupera; `/new`, `/reset` y `sessions.reset` eliminan inmediatamente las sustituciones de origen automático.
- Las respuestas al usuario anuncian una vez por cada cambio de estado las transiciones al respaldo y la recuperación al eliminarse el respaldo. Los turnos repetidos con el mismo par seleccionado/activo no repiten el aviso.
- `/status` muestra el modelo seleccionado y, cuando el estado del respaldo difiere, el modelo de respaldo activo y el motivo.
- La reconciliación de la sesión en directo da prioridad a las sustituciones persistidas de la sesión sobre los campos obsoletos del modelo en tiempo de ejecución.
- Si un error de cambio en directo apunta a un candidato posterior de la cadena de respaldos activa, OpenClaw salta directamente a ese modelo seleccionado en lugar de recorrer primero candidatos no relacionados.

La ejecución activa transporta directamente el candidato elegido. La reconciliación en directo solo cambia ese candidato por un cambio explícito pendiente del usuario, por lo que no se necesita ninguna sustitución temporal del respaldo ni reversión.

## Observabilidad y resúmenes de fallos

`runWithModelFallback(...)` registra los detalles de cada intento que alimentan los registros y los mensajes de tiempo de espera dirigidos al usuario:

- proveedor/modelo intentado
- motivo (`rate_limit`, `overloaded`, `billing`, `auth`, `model_not_found` y motivos similares de conmutación por error)
- estado/código opcional
- resumen del error legible para humanos

Los registros estructurados `model_fallback_decision` también incluyen campos `fallbackStep*` planos cuando un candidato falla, se omite o un respaldo posterior tiene éxito. Estos campos hacen explícita la transición intentada (`fallbackStepFromModel`, `fallbackStepToModel`, `fallbackStepFromFailureReason`, `fallbackStepFromFailureDetail`, `fallbackStepFinalOutcome`) para que los exportadores de registros y diagnósticos puedan reconstruir el fallo principal aunque también falle el respaldo final.

Cuando todos los candidatos fallan, OpenClaw genera `FallbackSummaryError`. El ejecutor externo de respuestas puede usarlo para crear un mensaje más específico, como "todos los modelos están temporalmente limitados por tasa", e incluir el vencimiento más próximo del periodo de espera cuando se conozca.

Ese resumen del periodo de espera tiene en cuenta el modelo:

- se ignoran los límites de tasa con ámbito de modelo no relacionados para la cadena de proveedor/modelo intentada
- si el bloqueo restante es un límite de tasa con ámbito de modelo coincidente, OpenClaw informa del último vencimiento coincidente que aún bloquea ese modelo

## Configuración relacionada

Consulte [Configuración del Gateway](/es/gateway/configuration) para:

- `auth.profiles` / `auth.order`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel` enrutamiento

Consulte [Modelos](/es/concepts/models) para obtener una descripción general más amplia de la selección de modelos y los mecanismos de respaldo.
