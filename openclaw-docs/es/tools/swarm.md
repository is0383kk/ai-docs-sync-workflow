---
read_when:
    - Quieres que un script de Code Mode distribuya el trabajo entre varios agentes
    - Necesita resultados secundarios estructurados, puntos de decisión o pipelines de primera finalización
    - Está habilitando o ajustando los límites de tools.swarm
    - Desea observar los procesos secundarios del recopilador en el panel de sesiones
sidebarTitle: Swarm
summary: Orquesta subagentes simultáneos desde scripts del modo Código con resultados estructurados, expansión en paralelo limitada y progreso en tiempo real
title: Enjambre
x-i18n:
    generated_at: "2026-07-26T05:58:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0bec17da7a2e144df35189a65d9b35d829815b545a4bb89652e6a681ca971a9
    source_path: tools/swarm.md
    workflow: 16
---

Swarm es una forma experimental y opcional de orquestar muchos subagentes desde un
script de [Modo de código](/es/tools/code-mode). Use el flujo de control normal de JavaScript o TypeScript,
como `Promise.all`, `while` y `if`, para distribuir el trabajo, recopilar
resultados y tomar decisiones.

No hay ningún DSL de grafos ni un formato de flujo de trabajo independiente. El programa es la
orquestación. Swarm añade a ese programa hijos recopiladores que se pueden esperar, resultados estructurados,
concurrencia limitada e informes de progreso.

## Habilitar Swarm

La ruta recomendada es **Settings → Labs → Swarm** en la interfaz de control. El
conmutador surte efecto inmediatamente y escribe `tools.swarm.enabled` en la
configuración.

También se puede habilitar Swarm directamente en `openclaw.json`:

```json5
{
  tools: {
    swarm: {
      enabled: true,
      maxConcurrent: 8,
      maxChildrenPerGroup: 50,
      maxTotalPerGroup: 200,
      waitTimeoutSecondsMax: 600,
      defaultAgentId: "",
    },
  },
}
```

La forma abreviada booleana habilita o deshabilita la función con todos los demás valores en
sus valores predeterminados:

```json5
{
  tools: {
    swarm: true,
  },
}
```

| Campo                   | Valor predeterminado | Descripción                                                                                                                    |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`               | `false` | Expone las opciones de creación en modo recopilador, `agents_wait` y la API invitada `agents.*` del Modo de código.                                   |
| `maxConcurrent`         | `8`     | Número máximo de hijos recopiladores que se ejecutan simultáneamente en un grupo de Swarm. Los hijos adicionales aceptados se ponen en cola en orden FIFO.          |
| `maxChildrenPerGroup`   | `50`    | Número máximo de hijos recopiladores activos en un grupo.                                                                                  |
| `maxTotalPerGroup`      | `200`   | Número máximo de hijos recopiladores que un grupo puede crear durante su vida útil. Este es el mecanismo de protección contra la creación descontrolada.                            |
| `waitTimeoutSecondsMax` | `600`   | Tiempo de espera máximo aceptado por una llamada a `agents_wait`. El valor predeterminado de la llamada es de 30 segundos.                                            |
| `defaultAgentId`        | `""`    | Agente de destino utilizado cuando una creación omite `agentId`. Un valor vacío usa el agente solicitante. Se aplican las listas de permitidos de subagentes existentes. |

Los valores numéricos deben ser enteros positivos. OpenClaw limita
`maxConcurrent` a `1`–`1000`, `maxChildrenPerGroup` a `1`–`10000`,
`maxTotalPerGroup` a `1`–`100000` y `waitTimeoutSecondsMax` a
`1`–`86400`.

Se puede reemplazar la configuración de Swarm para un agente configurado con
`agents.entries.*.tools.swarm`. El objeto por agente se combina sobre el objeto de nivel superior
`tools.swarm`.

## Requisitos

Las variables globales invitadas `agents.run`, `phase` y `log` requieren tanto Swarm como
el Modo de código de OpenClaw:

```json5
{
  tools: {
    codeMode: true,
    swarm: true,
  },
}
```

El Modo de código también debe tener acceso efectivo a `sessions_spawn`. Los perfiles de herramientas,
la política de permisos y denegaciones, las reglas del proveedor y la política del entorno aislado pueden eliminar esa herramienta.
Consulte [Activación del Modo de código](/es/tools/code-mode#activation) y
[Subagentes](/es/tools/subagents) si un script informa de que `sessions_spawn` no está
disponible.

Los valores `defaultAgentId` y `agentId` por ejecución deben nombrar un destino configurado
permitido por la política `subagents.allowAgents` del solicitante. OpenClaw rechaza
un destino desconocido o no permitido en lugar de recurrir a otro agente.

## Escribir un script de Swarm

Cuando Swarm está habilitado, el Modo de código expone esta API invitada:

```typescript
type AgentRunOptions = {
  label?: string;
  model?: string;
  thinking?: string;
  fastMode?: boolean | "auto";
  agentId?: string;
  schema?: Record<string, unknown>;
  phase?: string;
};

agents.run(prompt: string, options?: AgentRunOptions & { schema?: undefined }): Promise<string>;
agents.run<T>(prompt: string, options: AgentRunOptions & { schema: Record<string, unknown> }): Promise<T>;
phase(title: string): void;
log(message: string): void;
```

Sin `schema`, `agents.run()` se resuelve con el texto final del hijo. Con un
esquema JSON, se resuelve con el valor enviado mediante la herramienta
`structured_output` del hijo. Un hijo con errores, finalizado, cuyo tiempo de espera se haya agotado o cuyo esquema no sea válido
rechaza la promesa con un `SwarmAgentError`. Consulte las declaraciones generadas exactas
y los patrones breves de orquestación en `API.read("agents.d.ts")`
dentro del Modo de código.

Use `label` para asignar al hijo un nombre reconocible en el panel y la barra lateral. Use
`phase` en las opciones para publicar una fase inmediatamente antes de que se inicie ese hijo,
o llame a `phase()` cuando varios hijos pertenezcan a la misma etapa.
`log()` publica una nota breve de progreso. Las llamadas de progreso son de ejecución y olvido;
no retrasan el script si la interfaz no está disponible.

### Distribuir en paralelo con resultados estructurados

Este ejemplo inicia un investigador por tema, espera a que todos terminen y, a continuación,
pide a un último hijo que sintetice sus informes estructurados:

```javascript
const reportSchema = {
  type: "object",
  properties: {
    finding: { type: "string" },
    evidence: { type: "array", items: { type: "string" } },
    confidence: { type: "number" },
  },
  required: ["finding", "evidence", "confidence"],
  additionalProperties: false,
};

const topics = ["authentication", "storage", "recovery"];
phase("Revisión independiente");

const reports = await Promise.all(
  topics.map((topic) =>
    agents.run(`Revisa la ruta ${topic}. Devuelve un hallazgo con pruebas.`, {
      label: `review-${topic}`,
      thinking: "high",
      fastMode: "auto",
      schema: reportSchema,
    }),
  ),
);

phase("Síntesis");
log(`Se recopilaron ${reports.length} informes independientes.`);

return await agents.run(
  `Concilia estos informes y explica los desacuerdos:\n${JSON.stringify(reports)}`,
  { label: "synthesis" },
);
```

`Promise.all` es el límite de distribución y recopilación. OpenClaw inicia hasta
`maxConcurrent` hijos para el grupo y pone el resto en cola en el orden de
envío.

El Modo de código limita por separado las llamadas simultáneas al puente invitado mediante
`tools.codeMode.maxPendingToolCalls` (valor predeterminado `16`, máximo `128`). Para grupos muy
grandes, inicie lotes limitados por debajo de ese límite y deje margen para
`phase()`, `log()` y las transiciones de espera de los hijos. `maxConcurrent` limita los hijos en
ejecución; no aumenta el límite de llamadas al puente invitado.

### Repetir según una puerta de decisión

Use un bucle `while` limitado cuando cada pasada determine si se
necesita otra:

```javascript
const gateSchema = {
  type: "object",
  properties: {
    ready: { type: "boolean" },
    reason: { type: "string" },
    nextAction: { type: "string" },
  },
  required: ["ready", "reason", "nextAction"],
  additionalProperties: false,
};

let pass = 0;
let decision = { ready: false, reason: "Sin comprobar", nextAction: "Revisar" };

while (!decision.ready && pass < 4) {
  pass += 1;
  phase(`Pasada de decisión ${pass}`);
  decision = await agents.run(
    `Comprueba si las pruebas de la versión están completas. Decisión anterior: ${JSON.stringify(decision)}`,
    {
      label: `release-gate-${pass}`,
      schema: gateSchema,
    },
  );
  log(decision.reason);
}

if (!decision.ready) {
  throw new Error(`La puerta sigue cerrada después de ${pass} pasadas: ${decision.nextAction}`);
}

return decision;
```

Limite siempre los bucles de decisión. `maxTotalPerGroup` es el mecanismo de protección final,
no un sustituto de una condición de parada clara.

### Procesar el primer hijo que termine

`agents.run()` devuelve una promesa normal, por lo que `Promise.race` puede reaccionar al
primer hijo del Modo de código. Para los arneses que llaman a las herramientas de nivel inferior,
`agents_wait` proporciona el mismo límite de primera finalización: devuelve el resultado en cuanto
se completa al menos una ejecución solicitada o cuando se agota el tiempo de espera limitado.
Consulte [Usar Swarm desde otros arneses](#use-swarm-from-other-harnesses) para ver el
bucle de vaciado completo.

## Comportamiento de los hijos recopiladores

Los hijos recopiladores son sesiones ordinarias y aisladas de subagentes con una ruta de
finalización diferente. Escriben un resultado duradero del recopilador para que el padre lo
espere, en lugar de anunciar o dirigir una respuesta de vuelta a la sesión principal.

El agente de destino se resuelve en este orden:

1. `agentId` en la llamada de creación o `agents.run()`.
2. `tools.swarm.defaultAgentId`.
3. El agente solicitante.

Un agente trabajador dedicado y ligero resulta útil cuando los hijos de Swarm necesitan una superficie de
herramientas más pequeña, un modelo más económico o una política de entorno aislado más estricta. OpenClaw no incluye
un identificador de agente `worker` integrado; configure uno antes de designarlo como predeterminado.
Refuerce ese trabajador con `tools.swarm: false` en su configuración por agente para que
pueda ser creado, pero no pueda iniciar Swarms desde sus propias sesiones de nivel superior:

```json5
{
  tools: { swarm: { enabled: true, defaultAgentId: "worker" } },
  agents: {
    list: [
      {
        id: "main",
        default: true,
        subagents: { allowAgents: ["worker"] },
      },
      { id: "worker", tools: { swarm: false } },
    ],
  },
}
```

Las aprobaciones de los recopiladores se deniegan de forma segura. Un hijo nunca abre una solicitud de aprobación
para el operador. Una acción de herramienta que requiera aprobación se deniega, y el hijo puede
informar de esa denegación en su resultado para que el script decida qué hacer a continuación.

Para la salida estructurada, OpenClaw añade una herramienta sintética `structured_output` al
hijo y valida su carga útil con el esquema JSON proporcionado. Una
carga útil no válida o ausente recibe un aviso correctivo. Si el reintento tampoco
se valida, la finalización del recopilador conserva el texto sin procesar del hijo, deja
`structured` sin definir e incluye `schemaError`. El resultado de bajo nivel `agents_wait`
expone esos campos para la lógica de recuperación explícita.

### Los hijos son hojas

Los hijos de Swarm son hojas de forma predeterminada. La protección universal
`agents.defaults.subagents.maxSpawnDepth` evita que un hijo cree
sus propios hijos con la profundidad predeterminada de `1`. El patrón de orquestación habitual consiste en
devolver el trabajo al padre, no crear más trabajo desde un hijo:

```javascript
const plan = await agents.run("Planifica este trabajo como tareas independientes.", {
  schema: {
    type: "object",
    properties: { tasks: { type: "array", items: { type: "string" } } },
    required: ["tasks"],
    additionalProperties: false,
  },
});
return await Promise.all(plan.tasks.map((task) => agents.run(task)));
```

Los subagentes anidados son una opción que el operador debe habilitar mediante
`agents.defaults.subagents.maxSpawnDepth` y no se recomiendan para Swarm.
Los límites de grupo, los presupuestos y la observabilidad presuponen grupos de recopiladores planos.

Cada hijo tiene un propietario de admisión. Los hijos de anuncio e interactivos usan
`agents.defaults.subagents.maxChildrenPerAgent` (valor predeterminado `5`) y no cuentan
los hijos recopiladores. Los hijos recopiladores usan únicamente `maxChildrenPerGroup` y
`maxTotalPerGroup`; no consumen el presupuesto de hijos por sesión. La protección de
profundidad de creación se sigue aplicando a ambos modos.

Tras la admisión, los hijos por encima de `maxConcurrent` se ponen en cola FIFO dentro de su grupo de Swarm,
anidado dentro del carril global de subagentes. Estas capas de concurrencia ponen el trabajo
en cola en lugar de rechazarlo. La creación de un recopilador que supere cualquiera de los límites del grupo
se rechaza con la clave de configuración correspondiente en el error.

## Observar un Swarm

Abra el panel de la sesión principal en la interfaz de control mientras haya un Swarm activo.
El widget de Swarm representa cada grupo de recopiladores activo como un punto por hijo con
estado en cola, en ejecución, completado o con errores. Las etiquetas aparecen en la información emergente de los puntos, por lo que las etiquetas
breves y estables facilitan la lectura de los Swarms más grandes.

La barra lateral de la sesión conserva el árbol normal de padre e hijos. Expanda la fila del padre
para inspeccionar un hijo recopilador o abrir su transcripción sin perder la jerarquía de
Swarm.

Los resultados de los recopiladores siguen disponibles para esperarlos hasta que se archiva su grupo. Cuando todos los
miembros alcanzan su fecha límite de retención, OpenClaw archiva los hijos del grupo
como un lote para que los enjambres completados no permanezcan en el árbol de sesiones activas.

## Usar Swarm desde otros arneses

Se puede usar Swarm sin el modo de código de OpenClaw. Sus herramientas principales son
independientes del arnés: inicie hijos recopiladores con
`sessions_spawn({ collect: true })` y procéselos con llamadas acotadas a `agents_wait`.

El modo de código de Codex expone automáticamente las herramientas dinámicas de OpenClaw aptas bajo
`tools.*`. No usa la API de invitado QuickJS de OpenClaw ni requiere
`tools.codeMode`, pero `tools.swarm` debe seguir habilitado. Las llamadas
`agents_wait` del arnés de Codex admiten el tiempo de espera completo de 600 segundos.

Con el entorno de ejecución de Codex compatible actualmente, los resultados de las herramientas dinámicas de OpenClaw llegan al
modo de código como texto JSON. Analice cada resultado antes de leer los campos. Codex también
serializa las llamadas a herramientas dinámicas, por lo que `Promise.all` no envía varias
llamadas a `sessions_spawn` simultáneamente. Inicie los recopiladores en un bucle acotado;
los hijos ya aceptados pueden seguir ejecutándose mientras se envían los inicios posteriores.

```javascript
function parseToolResult(value) {
  if (typeof value !== "string") return value;
  return JSON.parse(value);
}

const tasks = [
  "Comprueba la ruta de autenticación.",
  "Comprueba la ruta de almacenamiento.",
  "Comprueba la ruta de recuperación.",
];
const launches = [];

for (const [index, task] of tasks.entries()) {
  const launch = parseToolResult(
    await tools.sessions_spawn({
      task,
      collect: true,
      label: `review-${index + 1}`,
    }),
  );
  if (launch.status !== "accepted") {
    throw new Error(launch.error ?? "No se aceptó el inicio del recopilador.");
  }
  launches.push(launch);
}

const pending = new Set(launches.map((launch) => launch.runId));
const completed = [];

while (pending.size > 0) {
  const ids = [...pending].slice(0, 1000);
  const batch = parseToolResult(
    await tools.agents_wait({
      ids,
      timeoutSeconds: 30,
    }),
  );

  // Rota esta ventana acotada detrás de los identificadores que aún no se hayan comprobado.
  for (const runId of ids) {
    if (pending.delete(runId)) pending.add(runId);
  }

  for (const item of batch.completed) {
    pending.delete(item.runId);
    if (item.status !== "done") {
      throw new Error(item.schemaError ?? item.result ?? `${item.runId}: ${item.status}`);
    }
    completed.push(item); // Procesa cada resultado en cuanto finalice.
  }

  for (const failure of batch.errors ?? []) {
    pending.delete(failure.runId);
    throw new Error(`${failure.runId}: ${failure.error}`);
  }
}

return completed;
```

Cada llamada a `agents_wait` acepta entre 1 y 1000 identificadores de ejecución. Devuelve:

```typescript
type AgentsWaitResult = {
  completed: Array<{
    runId: string;
    status: "done" | "failed" | "killed" | "timeout";
    result: string;
    structured?: unknown;
    schemaError?: string;
    sessionKey: string;
    label?: string;
    usage?: { inputTokens: number; outputTokens: number };
  }>;
  pending: string[];
  errors?: Array<{
    runId: string;
    error: "not_found" | "not_owner";
  }>;
};
```

La llamada devuelve el resultado inmediatamente cuando cualquier hijo solicitado ya ha finalizado,
cuando finaliza al menos un hijo pendiente, cuando no quedan identificadores pendientes válidos
o cuando vence su tiempo de espera. Los registros completados son idempotentes, por lo que pasar un
identificador de ejecución ya completado vuelve a devolver su resultado. Solo la sesión que lo inició
o su cadena de padres autorizados puede esperar a un recopilador.

Se trata de sondeo largo acotado, no de un bucle de consulta de estado intensivo. Siga pasando únicamente los
identificadores de ejecución restantes hasta que `pending` esté vacío. El modo recopilador admite subagentes
nativos de OpenClaw; no admite el entorno de ejecución ACP, la vinculación a hilos, las sesiones
visibles ni el modo de sesión persistente.

## Límites y hoja de ruta

Swarm v1 ejecuta hijos recopiladores de una sola ejecución; la API `agents.session()` prevista
añadirá trabajadores con estado y varios turnos. Actualmente, los hijos se ejecutan en el
carril de subagentes del Gateway local; la ubicación en la nube está prevista como una opción
explícita de inicio. Las definiciones de flujo de trabajo guardadas y un DSL de grafos no forman parte de la
dirección actual de Swarm.

## Contenido relacionado

- [Modo de código](/es/tools/code-mode) para conocer el entorno de ejecución de invitado QuickJS y las reglas de activación
- [Subagentes](/es/tools/subagents) para conocer la política de los hijos, el aislamiento y el comportamiento de las sesiones
- [Herramientas de entorno aislado multiagente](/es/tools/multi-agent-sandbox-tools) para conocer las restricciones por agente
- [Descripción general de las herramientas](/es/tools) para conocer los perfiles de herramientas y el enrutamiento de políticas
