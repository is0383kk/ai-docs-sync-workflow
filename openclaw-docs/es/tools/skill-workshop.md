---
read_when:
    - Quieres que el agente cree o actualice una skill desde el chat
    - Es necesario revisar, aplicar, rechazar o poner en cuarentena un borrador de skill generado
    - Está configurando la aprobación, la autonomía, el almacenamiento o los límites de Skill Workshop
    - Quiere saber dónde se revisan las propuestas de autoaprendizaje
sidebarTitle: Skill Workshop
summary: Crear y actualizar Skills del espacio de trabajo mediante la revisión de Skill Workshop
title: Taller de Skills
x-i18n:
    generated_at: "2026-07-26T04:57:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c2590f2a1bcad3b22ef8504eac7b3a44611c3fedc0df3832660f8926ce04252
    source_path: tools/skill-workshop.md
    workflow: 16
---

Skill Workshop es la vía gobernada de OpenClaw para crear y actualizar Skills
del espacio de trabajo. Los agentes y operadores nunca escriben `SKILL.md` directamente mediante esta
vía: crean una **propuesta** (un borrador pendiente con contenido, vinculación
de destino, estado del escáner, hashes y metadatos de reversión) que solo se convierte en una
Skill activa cuando se aplica.

Skill Workshop solo escribe Skills del espacio de trabajo. Nunca modifica Skills
incluidas, de plugins, de ClawHub, de raíces adicionales, administradas, de agentes personales ni del sistema.

## Cómo funciona

- **Primero la propuesta:** el contenido generado se almacena como `PROPOSAL.md`, no como
  `SKILL.md`.
- **Aplicar es la única escritura activa:** crear, actualizar y revisar nunca cambian
  las Skills activas.
- **Limitado al espacio de trabajo:** las creaciones se dirigen a la raíz `skills/` del espacio de trabajo; las actualizaciones
  solo se permiten para Skills editables del espacio de trabajo.
- **Sin sobrescritura:** la creación falla si la Skill de destino ya existe.
- **Vinculación mediante hash:** las propuestas de actualización se vinculan al hash actual del destino y pasan a
  `stale` si la Skill activa cambia antes de aplicarlas.
- **Sujeto al escáner:** antes de escribir, la aplicación vuelve a ejecutar el escáner de seguridad.
- **Recuperable:** la aplicación escribe los metadatos de reversión antes de modificar los archivos activos.
- **Superficies coherentes:** el chat, la CLI y el Gateway llaman al mismo servicio.

## Ciclo de vida

```text
crear/actualizar -> pendiente
revisar          -> pendiente
aplicar          -> aplicada
rechazar         -> rechazada
poner en cuarentena -> en cuarentena
cambio del destino -> obsoleta
```

Solo se puede revisar, aplicar, rechazar o poner en cuarentena una propuesta `pending`.

## Conservación del ciclo de vida

El Gateway registra el uso agregado de las Skills en la base de datos de estado compartida. Una vez al
día, revisa las Skills creadas y aplicadas mediante Skill Workshop. Las Skills que no se hayan utilizado durante
más de 30 días pasan a `stale`; después de 90 días pasan a `archived` y se
excluyen de las nuevas instantáneas de Skills de los agentes. Los archivos de las Skills archivadas permanecen intactos en
el disco. Las Skills creadas manualmente nunca se someten a conservación; solo las Skills creadas mediante propuestas de Skill
Workshop entran en la conservación del ciclo de vida.

Las Skills fijadas omiten las transiciones del ciclo de vida. Una Skill obsoleta vuelve a `active`
después de utilizarse y cuando se ejecuta el siguiente barrido. Las Skills archivadas solo vuelven mediante una
restauración explícita:

Las transiciones y restauraciones del ciclo de vida se aplican a las sesiones nuevas; las sesiones en ejecución conservan
su instantánea actual de Skills.

```bash
openclaw skills curator status
openclaw skills curator pin <skill>
openclaw skills curator unpin <skill>
openclaw skills curator restore <skill>
```

Todos los comandos del conservador aceptan `--json`. El estado también informa de candidatos de solapamiento deterministas
únicamente como sugerencias; nunca fusiona Skills ni llama a un modelo.

## Chat

Solicite al agente la Skill que desea; este llama a `skill_workshop` y devuelve un
identificador de propuesta.

### Aprender del trabajo reciente

Use `/learn` para convertir la conversación actual o las fuentes indicadas en una
propuesta de Skill guiada por estándares:

```text
/learn
/learn docs/runbook.md and https://example.com/guide; centrarse en la recuperación
```

Sin una solicitud, `/learn` pide al agente que extraiga el flujo de trabajo reutilizable de
la conversación actual. Con una solicitud, el agente trata las rutas, las URL, las notas
pegadas y las referencias a la conversación como fuentes, respetando los requisitos de enfoque, alcance y
nomenclatura. Recopila las fuentes con sus herramientas existentes y después llama a
`skill_workshop` con `action: "create"`.

La propuesta resultante permanece `pending`; `/learn` nunca la aplica. Revísela y
aplíquela mediante el flujo de aprobación normal o con `openclaw skills workshop`.

Crear:

```text
Crea una Skill llamada morning-catchup que ejecute mi rutina de los lunes para la bandeja de entrada.
```

Actualizar una Skill existente del espacio de trabajo:

```text
Actualiza trip-planning para que también compruebe los mapas de asientos antes de reservar.
```

Iterar sobre una propuesta pendiente:

```text
Muéstrame la propuesta morning-catchup.
Revísala para que también marque todo lo que se haya señalado como urgente.
Aplica la propuesta morning-catchup.
```

Las acciones `apply`, `reject` y `quarantine` iniciadas por el agente se ejecutan sin una solicitud
de aprobación adicional de forma predeterminada. Establezca `skills.workshop.approvalPolicy` en `"pending"`
para exigir la aprobación del operador antes de esas acciones.

Cuando se requiere aprobación, la solicitud identifica el id. de la propuesta y la Skill
de destino, y muestra la descripción de la propuesta, la cantidad de archivos auxiliares y el tamaño del cuerpo.
Las solicitudes de aprobación están limitadas para finalizar antes del mecanismo de vigilancia de la herramienta del agente. Si no
se recibe una decisión antes de que caduque la solicitud, la acción del ciclo de vida no se ejecuta:
la propuesta permanece pendiente y sin cambios. Decida más adelante en la interfaz de Skill Workshop o ejecute
`openclaw skills workshop apply|reject|quarantine <proposal-id>`. Los agentes no deben
reintentar en bucle una acción del ciclo de vida caducada.

## CLI

```bash
# Crear
openclaw skills workshop propose-create \
  --name morning-catchup \
  --description "Puesta al día diaria de la bandeja de entrada: clasificar, archivar, destacar, redactar, planificar" \
  --proposal ./PROPOSAL.md

# Actualizar una Skill existente del espacio de trabajo
openclaw skills workshop propose-update trip-planning --proposal ./PROPOSAL.md

# Enumerar e inspeccionar
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>

# Revisar antes de la aprobación
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md

# Cerrar
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Duplicada"
openclaw skills workshop quarantine <proposal-id> --reason "Requiere una revisión de seguridad"
```

Cada subcomando acepta `--agent <id>` (espacio de trabajo de destino; de forma predeterminada,
primero el inferido a partir del directorio de trabajo actual y después el agente predeterminado) y `--json` (salida estructurada).
`propose-create`, `propose-update` y `revise` también aceptan `--goal <text>` y
`--evidence <text>` para registrar el contexto de la propuesta junto con `--proposal`.

## Contenido de la propuesta

Mientras está pendiente, la propuesta se almacena como `PROPOSAL.md` con frontmatter exclusivo
de la propuesta:

```markdown
---
name: "morning-catchup"
description: "Puesta al día diaria de la bandeja de entrada: clasificar, archivar, destacar, redactar, planificar"
status: proposal
version: "v1"
date: "2026-05-30T00:00:00.000Z"
---
```

Al aplicarla, Skill Workshop escribe el `SKILL.md` activo y elimina los
campos exclusivos de la propuesta: `status`, `version` de la propuesta y `date` de la propuesta.

## Archivos auxiliares

Use `--proposal-dir` cuando la Skill propuesta necesite archivos junto a
`PROPOSAL.md`:

```bash
openclaw skills workshop propose-create \
  --name weekly-update \
  --description "Resumen del viernes: estadísticas, aspectos destacados y los tres puntos principales de la próxima semana" \
  --proposal-dir ./weekly-update-proposal
```

El directorio debe contener `PROPOSAL.md`. Los archivos auxiliares deben encontrarse en
`assets/`, `examples/`, `references/`, `scripts/` o `templates/`. Skill
Workshop los analiza, calcula sus hashes y los almacena con la propuesta; después los escribe
junto al `SKILL.md` activo únicamente al aplicar la propuesta.

Rutas de archivos auxiliares rechazadas: rutas absolutas, segmentos de ruta ocultos,
recorrido de rutas, rutas solapadas, archivos ejecutables, texto que no sea UTF-8, bytes nulos
y rutas fuera de las carpetas auxiliares estándar.

## Herramienta del agente

El modelo usa `skill_workshop` con un `action` obligatorio:
`create | update | revise | list | inspect | apply | reject | quarantine`.
Los demás parámetros se aplican según la acción:

| Parámetro                  | Utilizado por                                         | Notas                                                                |
| -------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------- |
| `name`                     | `create`, `inspect`, `revise`                        | Obligatorio para `create`; de lo contrario, resuelve una propuesta pendiente por nombre |
| `description`              | `create`, `update`, `revise`                         | Máximo de 160 bytes                                                   |
| `skill_name`               | `update`                                             | Nombre o clave de la Skill existente                                 |
| `proposal_content`         | `create`, `update`, `revise`                         | Se almacena como `PROPOSAL.md`; limitado por `skills.workshop.maxSkillBytes` |
| `support_files`            | `create`, `update`, `revise`                         | Matriz de `{ path, content }`                                         |
| `goal`, `evidence`         | `create`, `update`, `revise`                         | Contexto de texto libre                                              |
| `proposal_id`              | `inspect`, `revise`, `apply`, `reject`, `quarantine` | Propuesta de destino                                                  |
| `reason`                   | `apply`, `reject`, `quarantine`                      | Opcional                                                             |
| `query`, `status`, `limit` | `list`                                               | Filtrar/paginar; `limit` máximo de 50, valor predeterminado de 20 |

Los agentes deben usar `skill_workshop` para el trabajo de Skills generado. No deben
crear ni modificar archivos de propuestas mediante `write`, `edit`, `exec`, comandos
del shell ni operaciones directas del sistema de archivos.

<Note>
`skill_workshop` es una herramienta integrada del agente y está incluida en
`tools.profile: "coding"`. Si una política más estricta la oculta, añada
`skill_workshop` a la lista `tools.allow` activa, o use
`tools.alsoAllow: ["skill_workshop"]` cuando el ámbito utilice un perfil sin un
`tools.allow` explícito. Las ejecuciones aisladas no construyen la herramienta
Skill Workshop del lado del host, por lo que las acciones de revisión de propuestas deben ejecutarse desde una sesión
normal del agente en el lado del host o desde la CLI.
</Note>

## Skills sugeridas

OpenClaw detecta instrucciones duraderas como «la próxima vez», «recuerda» y correcciones reactivas
cuando finaliza un turno interactivo, incluidos los turnos fallidos. En el siguiente turno, el agente ofrece guardar
el flujo de trabajo detectado más reciente mediante `skill_workshop`; el usuario decide si crea una
propuesta. Esta sugerencia integrada no crea ni modifica por sí sola una Skill. Active
`skills.workshop.autonomous.enabled` para crear directamente propuestas pendientes. En la interfaz de
Control, la pestaña Workshop ofrece la misma opción como un interruptor **Autoaprendizaje** en el encabezado de la página y
como un botón de activación en el tablero de propuestas vacío.

### Analizar sesiones anteriores

La interfaz de Control puede revisar trabajos anteriores sin activar el autoaprendizaje autónomo.
Abra **Plugins → Workshop** y seleccione **Buscar ideas para Skills**. El análisis comienza con
las sesiones aptas más recientes y revisa una ventana limitada de trabajo sustancial.
Omite las sesiones de cron, Heartbeat, hooks, subagentes, ACP, propiedad de plugins y revisión
interna, además de las conversaciones con menos de seis turnos del modelo.

El revisor utiliza el modelo configurado del agente seleccionado y recibe un paquete de transcripciones
con los secretos censurados y el tamaño limitado. Aplica el mismo criterio conservador
que la revisión de experiencias: un patrón de recuperación concreto o un procedimiento estable que
eliminaría al menos dos llamadas futuras al modelo o a herramientas. El trabajo rutinario y los hechos
aislados no deben generar ninguna propuesta.

Un análisis puede crear o revisar como máximo tres propuestas pendientes. No puede aplicar,
rechazar, poner en cuarentena ni editar una Skill activa. Workshop muestra la cobertura acumulada;
por ejemplo, **20 sesiones revisadas · 18 de jun.–hoy · 2 ideas encontradas**. Seleccione
**Analizar trabajo anterior** para continuar desde el cursor persistente de la sesión más antigua. Después de
agotar el historial disponible, la acción pasa a ser **Analizar trabajo nuevo**.

La revisión histórica es manual incluso cuando
`skills.workshop.autonomous.enabled` es `false`. Cada clic inicia una ejecución del modelo,
por lo que se aplican los precios y las condiciones de tratamiento de datos del proveedor. El cursor y los recuentos de cobertura
se almacenan en la base de datos de estado compartida de OpenClaw; el contenido de la transcripción no se copia
en el estado del análisis.

Con la captura autónoma habilitada, OpenClaw también puede realizar una revisión conservadora después de un trabajo
sustancial y satisfactorio y una vez que todo el sistema de agentes queda inactivo. Esa revisión aislada puede crear o
revisar como máximo una propuesta pendiente. No puede actualizar una skill activa ni aplicar, rechazar o poner en cuarentena una
propuesta, incluso cuando `approvalPolicy` es `"auto"`.

Consulte [Autoaprendizaje](/es/tools/self-learning) para obtener información sobre la habilitación, los requisitos, la privacidad y los costes,
el umbral de las propuestas y la solución de problemas.

## Aprobación y autonomía

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: false,
      },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
  },
}
```

| Configuración                    | Valor predeterminado  | Efecto                                                                                                                                                              |
| -------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `autonomous.enabled`       | `false`  | Crea propuestas pendientes a partir de correcciones explícitas y, tras un periodo de inactividad, de trabajo sustancial completado que permita una recuperación reutilizable o un ahorro significativo de recorridos de ida y vuelta.   |
| `allowSymlinkTargetWrites` | `false`  | Permite que la aplicación escriba a través de enlaces simbólicos de skills del espacio de trabajo cuyo destino real figure en `skills.load.allowSymlinkTargets`.                                                 |
| `approvalPolicy`           | `"auto"` | `"auto"` omite una solicitud de confirmación adicional para las acciones `apply`, `reject` o `quarantine` iniciadas por el agente (el agente aún debe invocar la acción). `"pending"` requiere aprobación. |
| `maxPending`               | `50`     | Limita las propuestas pendientes y en cuarentena por espacio de trabajo (1-200).                                                                                                       |
| `maxSkillBytes`            | `40000`  | Limita el tamaño del cuerpo de la propuesta en bytes (1024-200000).                                                                                                                     |

La captura autónoma reconoce reglas prospectivas (por ejemplo, «a partir de ahora») y
correcciones reactivas (por ejemplo, «eso no es lo que pedí»). Agrupa las instrucciones nuevas por tema en un máximo
de tres propuestas por turno, dirige las coincidencias de vocabulario a skills existentes con permisos de escritura en el espacio de trabajo y
revisa su propia propuesta pendiente cuando otra corrección se dirige a la misma skill.

Para un trabajo sustancial completado satisfactoriamente sin una corrección explícita, una ejecución aislada del modelo
seleccionado decide si la trayectoria completada supera el umbral conservador de propuestas. No se solicita
al modelo en primer plano que aprenda antes de responder. El revisor en segundo plano conserva la
ejecución en primer plano como procedencia de la propuesta, no puede acceder a las herramientas generales del agente y no puede tomar decisiones
sobre el ciclo de vida. La revisión solo comienza cuando el entorno de ejecución en primer plano informa tanto de su modelo resuelto exacto
como de que `skill_workshop` estaba realmente disponible. Por lo tanto, una política de herramientas restrictiva o desconocida
impide la operación de forma segura y no crea ninguna propuesta.

Consulte [Autoaprendizaje](/es/tools/self-learning) para conocer el comportamiento completo de la revisión autónoma y el modelo
de seguridad.

Las descripciones de las propuestas siempre están limitadas a 160 bytes, independientemente de
`maxSkillBytes`.

## Métodos del Gateway

| Método                             | Ámbito            |
| ---------------------------------- | ---------------- |
| `skills.proposals.list`            | `operator.read`  |
| `skills.proposals.inspect`         | `operator.read`  |
| `skills.proposals.historyStatus`   | `operator.read`  |
| `skills.proposals.historyScan`     | `operator.admin` |
| `skills.proposals.create`          | `operator.admin` |
| `skills.proposals.update`          | `operator.admin` |
| `skills.proposals.revise`          | `operator.admin` |
| `skills.proposals.requestRevision` | `operator.admin` |
| `skills.proposals.apply`           | `operator.admin` |
| `skills.proposals.reject`          | `operator.admin` |
| `skills.proposals.quarantine`      | `operator.admin` |
| `skills.curator.status`            | `operator.read`  |
| `skills.curator.pin`               | `operator.admin` |
| `skills.curator.unpin`             | `operator.admin` |
| `skills.curator.restore`           | `operator.admin` |

`requestRevision` solo está disponible en el Gateway (no existe un equivalente en la CLI ni en las herramientas del agente):
reenvía instrucciones de revisión en texto libre a la sesión de chat del agente propietario
en lugar de sustituir directamente `PROPOSAL.md`, para las interfaces de usuario que piden al agente que
revise en vez de enviar contenido nuevo literal.

`historyStatus` y `historyScan` son métodos auxiliares de la interfaz de control. `historyScan`
acepta `direction: "older" | "newer"`; siempre deja los resultados como
propuestas pendientes.

## Almacenamiento

```text
<OPENCLAW_STATE_DIR>/skill-workshop/
  proposals.json
  proposals/<proposal-id>/
    proposal.json
    PROPOSAL.md
    rollback.json
    assets/
    examples/
    references/
    scripts/
    templates/
```

Directorio de estado predeterminado: `~/.openclaw`.

- `proposal.json`: registro canónico de la propuesta.
- `proposals.json`: índice para listados rápidos, reconstruible a partir de las carpetas de propuestas.
- `PROPOSAL.md`: propuesta de skill pendiente.
- `rollback.json`: metadatos de recuperación escritos antes de que la aplicación modifique los archivos activos.

## Límites

| Límite                           | Valor                                                                |
| ------------------------------- | -------------------------------------------------------------------- |
| Descripción                     | 160 bytes                                                            |
| Cuerpo de la propuesta                   | `skills.workshop.maxSkillBytes` (valor predeterminado: 40,000; límite máximo: 1 MiB) |
| Archivos auxiliares                   | 64 por propuesta                                                      |
| Tamaño de los archivos auxiliares               | 256 KiB cada uno, 2 MiB en total                                            |
| Propuestas pendientes + en cuarentena | `skills.workshop.maxPending` por espacio de trabajo (valor predeterminado: 50)              |

## Solución de problemas

| Problema                                        | Solución                                                                                                                                                                                                  |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Skill proposal description is too large`      | Reduzca `description` a 160 bytes o menos.                                                                                                                                                                 |
| `Skill proposal content is too large`          | Reduzca el cuerpo de la propuesta o aumente `skills.workshop.maxSkillBytes`.                                                                                                                                         |
| `Target skill changed after proposal creation` | Revise la propuesta con respecto al destino actual o cree una propuesta nueva.                                                                                                                                   |
| `Proposal scan failed`                         | Examine los hallazgos del analizador y, a continuación, revise o ponga en cuarentena la propuesta.                                                                                                                                           |
| `untrusted symlink target`                     | Configure `skills.load.allowSymlinkTargets` y habilite `skills.workshop.allowSymlinkTargetWrites` únicamente para raíces de skills compartidas de forma intencional.                                                                  |
| `Support file paths must be under one of...`   | Mueva los archivos auxiliares a `assets/`, `examples/`, `references/`, `scripts/` o `templates/`.                                                                                                                |
| La propuesta no aparece en la lista                 | Compruebe el espacio de trabajo `--agent` seleccionado y `OPENCLAW_STATE_DIR`.                                                                                                                                            |
| El agente no puede invocar `skill_workshop`             | Compruebe la política de herramientas activa y el modo de ejecución. `coding` incluye la herramienta; las políticas `tools.allow` restrictivas deben incluirla explícitamente, y las ejecuciones en un entorno aislado deben utilizar una sesión normal del agente en el host o la CLI. |

### Diagnóstico de la política de herramientas

Cuando la captura autónoma está habilitada, `openclaw doctor` ejecuta la
comprobación `core/doctor/skill-workshop-tool-policy` para el agente predeterminado. Si la política
oculta `skill_workshop`, la advertencia indica la primera capa de configuración que lo excluye y
el cambio exacto que debe realizarse en `allow` o `alsoAllow`. Los manuales operativos antiguos todavía pueden usar
`openclaw plugins inspect skill-workshop`; ahora ese comando explica que Skill
Workshop está integrado y muestra la misma indicación sobre la política cuando corresponde.

## Contenido relacionado

- [Skills](/es/tools/skills) para conocer el orden de carga, la precedencia y la visibilidad
- [Autoaprendizaje](/es/tools/self-learning) para conocer las propuestas conservadoras de skills posteriores a la ejecución
- [Creación de skills](/es/tools/creating-skills) para conocer los fundamentos de `SKILL.md`
  escritas a mano
- [Configuración de Skills](/es/tools/skills-config) para consultar el esquema completo de `skills.workshop`
- [CLI de Skills](/es/cli/skills) para consultar los comandos de `openclaw skills`
