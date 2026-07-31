---
read_when:
    - Configurar el comportamiento de carga, instalación o restricción de Skills
    - Configuración de la visibilidad de Skills por agente
    - Ajuste de los límites o la política de aprobación de Skill Workshop
sidebarTitle: Skills config
summary: Referencia completa del esquema de configuración `skills.*`, las listas de permitidos de agentes, los ajustes del taller y la gestión de variables de entorno del entorno aislado.
title: Configuración de Skills
x-i18n:
    generated_at: "2026-07-26T04:56:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bc154bdf8a8537095a4d39bc6e86ebfd716e6beacd45def9c8a1c15fcdc93698
    source_path: tools/skills-config.md
    workflow: 16
---

La mayor parte de la configuración de Skills se encuentra en `skills` dentro de
`~/.openclaw/openclaw.json`. La visibilidad específica de cada agente se encuentra en
`agents.defaults.skills` y `agents.entries.*.skills`.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
      watch: true,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm",
      allowUploadedArchives: false,
    },
    workshop: {
      autonomous: { enabled: false },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<Note>
  Para la generación de imágenes integrada, use `agents.defaults.mediaModels.image`
  junto con la herramienta principal `image_generate` en lugar de `skills.entries`. Las entradas de
  Skills son únicamente para flujos de trabajo de Skills personalizados o de terceros.
</Note>

## Carga (`skills.load`)

<ParamField path="skills.load.extraDirs" type="string[]">
  Directorios adicionales de Skills que se examinarán, con la menor prioridad (por debajo de
  Skills integrados y de plugins). Las rutas se expanden con compatibilidad con `~`.
</ParamField>

<ParamField path="skills.load.allowSymlinkTargets" type="string[]">
  Directorios de destino reales y de confianza a los que pueden resolverse las carpetas de Skills
  con enlaces simbólicos, incluso cuando el enlace simbólico se encuentra fuera de la raíz configurada. Use esta opción para
  estructuras intencionadas de repositorios adyacentes, como
  `<workspace>/skills/manager -> ~/Projects/manager/skills`. Mantenga esta lista
  limitada; no señale raíces amplias como `~` o `~/Projects`.
</ParamField>

<ParamField path="skills.load.watch" type="boolean" default="true">
  Supervisa las carpetas de Skills y actualiza la instantánea de Skills cuando cambian los archivos
  `SKILL.md`. Incluye los archivos anidados bajo raíces de Skills agrupadas.
</ParamField>

## Instalación (`skills.install`)

<ParamField path="skills.install.preferBrew" type="boolean" default="true">
  Da preferencia a los instaladores de Homebrew cuando `brew` está disponible.
</ParamField>

<ParamField path="skills.install.nodeManager" type='"npm" | "pnpm" | "yarn" | "bun"' default='"npm"'>
  Preferencia de gestor de paquetes de Node para las instalaciones de Skills. Esto solo afecta a las
  instalaciones de Skills: la CLI de OpenClaw y el entorno de ejecución del Gateway requieren Node porque el
  almacén de estado canónico usa `node:sqlite`. `openclaw setup --node-manager` y
  `openclaw onboard --node-manager` aceptan `npm`, `pnpm` o `bun`; establezca
  `"yarn"` directamente en la configuración para las instalaciones de Skills basadas en Yarn.
</ParamField>

<ParamField path="skills.install.allowUploadedArchives" type="boolean" default="false">
  Permite que los clientes de confianza del Gateway `operator.admin` instalen archivos zip
  privados preparados mediante `skills.upload.*`. Las instalaciones normales de ClawHub no
  necesitan esta opción.
</ParamField>

## Política de instalación del operador (`security.installPolicy`)

Use `security.installPolicy` cuando los operadores necesiten un comando local de confianza para
aprobar o bloquear instalaciones de Skills y plugins mediante una política específica del host. La
política se ejecuta después de que OpenClaw haya preparado el material de origen y antes de que la instalación
o actualización continúe. Se aplica a Skills de ClawHub, Skills cargadas, Skills de Git/locales,
instaladores de dependencias de Skills y orígenes de instalación/actualización de plugins.

```json5
{
  security: {
    installPolicy: {
      enabled: true,
      // Omita targets para incluir todos los destinos compatibles.
      targets: ["skill", "plugin"],
      exec: {
        source: "exec",
        command: "/usr/local/bin/openclaw-install-policy",
        args: ["--json"],
        timeoutMs: 10000,
        noOutputTimeoutMs: 10000,
        maxOutputBytes: 1048576,
        passEnv: ["OPENCLAW_STATE_DIR", "PATH"],
        env: { POLICY_MODE: "strict" },
        trustedDirs: ["/usr/local/bin"],
      },
    },
  },
}
```

<ParamField path="security.installPolicy.enabled" type="boolean" default="false">
  Habilita la política de instalación propiedad del operador. Cuando se habilita sin un comando
  `exec` válido, las instalaciones se bloquean de forma predeterminada.
</ParamField>

<ParamField path="security.installPolicy.targets" type='("skill" | "plugin")[]'>
  Filtro de destinos opcional. Cuando se omite, la política se aplica a todos los
  destinos compatibles para que las nuevas instalaciones no se permitan de forma inesperada.
</ParamField>

<ParamField path="security.installPolicy.exec.command" type="string">
  Ruta absoluta al ejecutable de política de confianza. OpenClaw lo ejecuta sin un
  shell y valida la ruta antes de usarla.
</ParamField>

<ParamField path="security.installPolicy.exec.args" type="string[]">
  Argumentos estáticos que se pasan después de `command`.
</ParamField>

<ParamField path="security.installPolicy.exec.timeoutMs" type="number" default="10000">
  Tiempo máximo de ejecución según el reloj para una decisión de política.
</ParamField>

<ParamField path="security.installPolicy.exec.noOutputTimeoutMs" type="number" default="timeoutMs">
  Tiempo máximo sin salida en stdout o stderr antes de que la política
  se bloquee de forma predeterminada.
</ParamField>

<ParamField path="security.installPolicy.exec.maxOutputBytes" type="number" default="1048576">
  Número máximo combinado de bytes de stdout y stderr aceptados del proceso de política.
</ParamField>

<ParamField path="security.installPolicy.exec.env" type="Record<string, string>">
  Variables de entorno literales proporcionadas al proceso de política.
</ParamField>

<ParamField path="security.installPolicy.exec.passEnv" type="string[]">
  Nombres de variables de entorno copiados del proceso de OpenClaw al
  proceso de política. Solo se pasan las variables indicadas.
</ParamField>

<ParamField path="security.installPolicy.exec.trustedDirs" type="string[]">
  Lista de permitidos opcional de directorios que pueden contener el ejecutable de política.
</ParamField>

<ParamField path="security.installPolicy.exec.allowInsecurePath" type="boolean" default="false">
  Omite las comprobaciones de propiedad y permisos de la ruta del comando. Úselo únicamente cuando la
  ruta esté protegida por otro mecanismo.
</ParamField>

<ParamField path="security.installPolicy.exec.allowSymlinkCommand" type="boolean" default="false">
  Permite que la ruta de comando configurada sea un enlace simbólico. El destino resuelto
  debe seguir satisfaciendo las demás comprobaciones de ruta. Los argumentos de scripts del intérprete deben
  ser archivos normales directos, no enlaces simbólicos.
</ParamField>

La política recibe un objeto JSON por stdin con `protocolVersion: 1`,
`openclawVersion`, `targetType`, `targetName`, `sourcePath`, `sourcePathKind`,
el campo estructurado opcional `source`, el campo estructurado `origin` y `request`. Debe
escribir un objeto JSON por stdout: `{ "protocolVersion": 1, "decision": "allow" }`
o `{ "protocolVersion": 1, "decision": "block", "reason": "..." }`. Una salida con código distinto de cero,
un tiempo de espera agotado, JSON con formato incorrecto, campos ausentes o versiones de protocolo
no compatibles provocan un bloqueo predeterminado.

OpenClaw no ejecuta la política de instalación durante el inicio normal del Gateway.
Las instalaciones y actualizaciones se bloquean de forma predeterminada cuando la política está habilitada pero no disponible.
`openclaw doctor` realiza una validación estática; `openclaw doctor --deep`
ejecuta una prueba de instalación sintética con el comando configurado.

Las actualizaciones masivas aplican la política a cada destino: una actualización bloqueada de una Skill o un plugin hace
que ese destino falle sin deshabilitar la política ni omitir los destinos posteriores del
lote.

Ejemplo de stdin:

```json
{
  "protocolVersion": 1,
  "openclawVersion": "2026.6.1",
  "targetType": "skill",
  "targetName": "weather",
  "sourcePath": "/var/folders/.../openclaw-skill-clawhub/root",
  "sourcePathKind": "directory",
  "source": {
    "kind": "clawhub",
    "authority": "openclaw",
    "mutable": false,
    "network": true
  },
  "origin": {
    "type": "clawhub",
    "registry": "https://clawhub.openclaw.ai",
    "slug": "weather",
    "version": "1.0.0"
  },
  "request": {
    "kind": "skill-install",
    "mode": "install",
    "requestedSpecifier": "clawhub:weather@1.0.0"
  },
  "skill": {
    "installId": "clawhub"
  }
}
```

Comando de política mínimo:

```js
#!/usr/bin/env node

let input = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => {
  input += chunk;
});
process.stdin.on("end", () => {
  const request = JSON.parse(input);
  if (request.targetType === "plugin" && request.source?.kind === "local-path") {
    process.stdout.write(
      JSON.stringify({
        protocolVersion: 1,
        decision: "block",
        reason: "las rutas de plugins locales no están aprobadas en este host",
      }),
    );
    return;
  }
  process.stdout.write(JSON.stringify({ protocolVersion: 1, decision: "allow" }));
});
```

## Lista de Skills integradas permitidas

<ParamField path="skills.allowBundled" type="string[]">
  Lista de permitidos opcional únicamente para Skills **integradas**. Cuando se establece, solo las Skills
  integradas incluidas en la lista son elegibles. Las Skills administradas, a nivel de agente y del espacio de trabajo
  no se ven afectadas.
</ParamField>

## Entradas por Skill (`skills.entries`)

De forma predeterminada, las claves bajo `entries` coinciden con el `name` de la Skill. Si una Skill define
`metadata.openclaw.skillKey`, use esa clave en su lugar. Escriba entre comillas los nombres con guiones
(JSON5 permite claves entre comillas).

<ParamField path="skills.entries.<key>.enabled" type="boolean">
  `false` deshabilita la Skill incluso cuando está integrada o instalada. La
  Skill integrada `coding-agent` es opcional: establézcala en `true` y asegúrese de que una de
  `claude`, `codex`, `opencode` u otra CLI compatible esté instalada y
  autenticada.
</ParamField>

<ParamField path="skills.entries.<key>.apiKey" type='string | { source, provider, id }'>
  Campo de conveniencia para Skills que declaran `metadata.openclaw.primaryEnv`.
  Admite una cadena de texto sin formato o una SecretRef: `{ source: "env", provider: "default", id: "VAR_NAME" }`.
</ParamField>

<ParamField path="skills.entries.<key>.env" type="Record<string, string>">
  Variables de entorno inyectadas para la ejecución del agente. Solo se inyectan cuando la
  variable aún no está establecida en el proceso.
</ParamField>

<ParamField path="skills.entries.<key>.config" type="object">
  Contenedor opcional para campos de configuración personalizados de cada Skill.
</ParamField>

## Listas de agentes permitidos (`agents`)

Use la configuración del agente cuando desee las mismas raíces de Skills de la máquina o del espacio de trabajo, pero un
conjunto de Skills visibles diferente para cada agente.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // base compartida
    },
    list: [
      { id: "writer" }, // hereda github, weather
      { id: "docs", skills: ["docs-search"] }, // reemplaza por completo los valores predeterminados
      { id: "locked-down", skills: [] }, // sin Skills
    ],
  },
}
```

<ParamField path="agents.defaults.skills" type="string[]">
  Lista de permitidos de referencia compartida que heredan los agentes que omiten
  `agents.entries.*.skills`. Omítala por completo para dejar las Skills sin restricciones de forma
  predeterminada.
</ParamField>

<ParamField path="agents.entries.*.skills" type="string[]">
  Conjunto final explícito de Skills para ese agente. Las listas explícitas **reemplazan**
  los valores predeterminados heredados; no se combinan. Establézcalo en `[]` para no exponer ninguna Skill a
  ese agente.
</ParamField>

<Warning>
  Las listas de Skills permitidas de los agentes son un filtro de visibilidad y carga para la detección de Skills de
  OpenClaw, los prompts, la detección de comandos con barra, la sincronización del entorno aislado y las
  instantáneas de Skills. No constituyen un límite de autorización durante la ejecución del shell. Si un agente
  puede ejecutar `exec` en el host, ese shell aún puede ejecutar clientes externos o leer
  archivos del host visibles para el usuario de ejecución, incluidos registros de clientes
  MCP como `~/.openclaw/skills/config/mcporter.json`. Para
  aislar MCP por agente, combine las listas de Skills permitidas con el aislamiento mediante entornos aislados o usuarios
  del sistema operativo, deniegue la ejecución en el host o restrínjala mediante una lista de permitidos estricta, y dé preferencia a credenciales
  por agente en el servidor MCP.
</Warning>

## Taller (`skills.workshop`)

<ParamField path="skills.workshop.autonomous.enabled" type="boolean" default="false">
  Cuando `true`, OpenClaw puede crear propuestas pendientes a partir de correcciones duraderas
  y puede revisar trabajos completados correctamente que sean sustanciales después de que el sistema quede
  inactivo. Esto puede añadir una ejecución del modelo en segundo plano después de los turnos aptos. La creación
  de Skills solicitada por el usuario y `/learn` siguen funcionando cuando el ajuste es `false`.
</ParamField>

Consulte [Autoaprendizaje](/es/tools/self-learning) para obtener información sobre los requisitos, la privacidad, el coste,
los permisos limitados a propuestas y la solución de problemas.

<ParamField path="skills.workshop.approvalPolicy" type='"pending" | "auto"' default='"auto"'>
  `auto` permite que el agente inicie la aplicación, el rechazo o la cuarentena sin una
  solicitud de aprobación adicional. `pending` requiere la aprobación del operador.
</ParamField>

<ParamField path="skills.workshop.allowSymlinkTargetWrites" type="boolean" default="false">
  Permite que la aplicación de Skill Workshop escriba a través de enlaces simbólicos de Skills del espacio de trabajo cuyo
  destino real ya sea de confianza según `skills.load.allowSymlinkTargets`. Mantenga
  esta opción desactivada salvo que la aplicación de propuestas generadas deba modificar esa raíz
  compartida de Skills.
</ParamField>

<ParamField path="skills.workshop.maxPending" type="number" default="50">
  Número máximo de propuestas pendientes y en cuarentena conservadas por espacio de trabajo (intervalo
  permitido: 1-200).
</ParamField>

<ParamField path="skills.workshop.maxSkillBytes" type="number" default="40000">
  Tamaño máximo del cuerpo de la propuesta en bytes (intervalo permitido: 1024-200000). Las descripciones
  de las propuestas tienen por separado un límite estricto de 160 bytes, porque aparecen
  en la salida de detección y de listado.
</ParamField>

Consulte [Skill Workshop](/es/tools/skill-workshop) para obtener información sobre el ciclo de vida de las propuestas, los comandos
de la CLI, los parámetros de las herramientas del agente y los métodos del Gateway que controla esta configuración.

## Raíces de Skills con enlaces simbólicos

De forma predeterminada, las raíces de Skills del espacio de trabajo, del agente del proyecto, de directorios adicionales e integradas son
límites de contención. Una carpeta de Skills con un enlace simbólico bajo `<workspace>/skills`
que se resuelva fuera de la raíz se omite y se registra un mensaje.

Para permitir una disposición intencionada de enlaces simbólicos, declare el destino de confianza:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

Con esta configuración, `<workspace>/skills/manager -> ~/Projects/manager/skills`
se acepta después de resolver la ruta real. `extraDirs` analiza directamente el repositorio
hermano; `allowSymlinkTargets` conserva la ruta con enlace simbólico para las disposiciones
existentes.

De forma predeterminada, la aplicación de Skill Workshop no escribe a través de esos enlaces simbólicos. Para
permitir que la aplicación de Workshop modifique Skills bajo destinos de enlaces simbólicos que ya sean de confianza, active
esta opción por separado:

```json5
{
  skills: {
    load: {
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    workshop: {
      allowSymlinkTargetWrites: true,
    },
  },
}
```

Los directorios administrados `~/.openclaw/skills` y personales `~/.agents/skills`
ya aceptan incondicionalmente enlaces simbólicos a directorios de Skills (la contención
`SKILL.md` por Skill sigue siendo aplicable); `allowSymlinkTargets` solo es necesario
para las raíces del espacio de trabajo, de directorios adicionales y del agente del proyecto (`<workspace>/.agents/skills`).

## Skills en entornos aislados y variables de entorno

<Warning>
  `skills.entries.<skill>.env` y `apiKey` solo se aplican a ejecuciones en el **host**.
  Dentro de un entorno aislado no tienen efecto: una Skill que dependa de
  `GEMINI_API_KEY` fallará con `apiKey not configured` a menos que la variable se
  proporcione por separado al entorno aislado.
</Warning>

Pase secretos a un entorno aislado de Docker mediante:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          env: { GEMINI_API_KEY: "your-key-here" },
        },
      },
    },
  },
}
```

<Note>
  Los usuarios con acceso al daemon de Docker pueden inspeccionar los valores de `sandbox.docker.env`
  mediante los metadatos de Docker. Use un archivo de secretos montado, una imagen personalizada u
  otra vía de entrega cuando esa exposición no sea aceptable.
</Note>

## Recordatorio del orden de carga

```text
workspace/skills      (más alta)
workspace/.agents/skills
~/.agents/skills
~/.openclaw/skills
Skills integradas
skills.load.extraDirs (más baja)
```

Los cambios en las Skills y en la configuración surten efecto en la siguiente sesión nueva cuando el
observador está activado, o en el siguiente turno del agente cuando el observador detecta un
cambio.

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Referencia de Skills" href="/es/tools/skills" icon="puzzle-piece">
    Qué son las Skills, su orden de carga, sus controles de acceso y el formato de SKILL.md.
  </Card>
  <Card title="Creación de Skills" href="/es/tools/creating-skills" icon="hammer">
    Creación de Skills personalizadas para el espacio de trabajo.
  </Card>
  <Card title="Skill Workshop" href="/es/tools/skill-workshop" icon="flask">
    Cola de propuestas para Skills redactadas por agentes.
  </Card>
  <Card title="Autoaprendizaje" href="/es/tools/self-learning" icon="brain">
    Propuestas conservadoras y opcionales a partir de trabajos completados.
  </Card>
  <Card title="Comandos con barra diagonal" href="/es/tools/slash-commands" icon="terminal">
    Catálogo nativo de comandos con barra diagonal y directivas de chat.
  </Card>
</CardGroup>
