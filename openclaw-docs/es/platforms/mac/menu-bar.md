---
read_when:
    - Ajuste de la interfaz del menú de Mac o de la lógica de estado
summary: Lógica de estado de la barra de menús y qué se muestra a los usuarios
title: Barra de menús
x-i18n:
    generated_at: "2026-07-26T05:11:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d53cd15109864b88010f41ccf4c46ea7fff6721bc6632630d83a558084cb2d62
    source_path: platforms/mac/menu-bar.md
    workflow: 16
---

## Qué se muestra

- El estado de trabajo actual del agente se muestra en el icono de la barra de menús y en la primera fila de estado del menú.
- El estado de salud se oculta mientras hay trabajo activo; vuelve cuando todas las sesiones están inactivas.
- Un elemento raíz «Contexto» abre un submenú con las sesiones recientes en lugar de expandirlas en el menú raíz.
- Un bloque «Nodos» en el menú raíz muestra únicamente los **dispositivos** emparejados (de `node.list`), no las entradas de cliente/presencia.
- Una sección raíz «Uso» aparece debajo de Contexto cuando hay instantáneas de uso del proveedor disponibles, seguidas de los detalles de costes cuando están disponibles.
- **Chat rápido** abre el editor flotante de la sesión principal; su atajo global actual aparece junto al elemento.

## Modelo de estado

- Fuente: `WorkActivityStore` (`apps/macos/Sources/OpenClaw/WorkActivityStore.swift`).
- Los eventos llegan como `ControlAgentEvent` con un `runId`; el controlador (`ControlChannel.routeWorkActivity`) lee `sessionKey` de la carga útil del evento y usa `"main"` de forma predeterminada si no está presente.
- Prioridad: la sesión principal (`sessionKey == "main"` de forma predeterminada) siempre prevalece. Si la sesión principal está activa, su estado se muestra inmediatamente. Si está inactiva, se muestra en su lugar la sesión no principal activa más recientemente. El almacén no cambia en mitad de la actividad; solo cambia cuando la sesión actual pasa a estar inactiva o la principal se activa.
- Tipos de actividad:
  - `job`: ejecución de comandos de alto nivel (`state: started|streaming|done|error|...`).
  - `tool`: `phase: start|result` con `name`, `meta`/`args` opcionales.

## Enumeración IconState (Swift)

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)` (anulación de depuración)

### ActivityKind -> símbolo de insignia

`ActivityKind` encapsula un `ToolKind` (`bash`, `read`, `write`, `edit`, `attach`, `other`) o un `job` independiente. Cada uno se asigna a una insignia de SF Symbols dibujada sobre el icono de la criatura (`IconState.badgeSymbolName`):

| Tipo            | Símbolo                            |
| --------------- | ---------------------------------- |
| `bash`          | `chevron.left.slash.chevron.right` |
| `read`          | `doc`                              |
| `write`         | `pencil`                           |
| `edit`          | `pencil.tip`                       |
| `attach`        | `paperclip`                        |
| `other` / `job` | `gearshape.fill`                   |

### Correspondencia visual

- `idle`: criatura normal, sin insignia.
- `workingMain`: insignia con símbolo, tinte completo (prominencia `.primary`), animación de patas «trabajando».
- `workingOther`: insignia con símbolo, tinte atenuado (prominencia `.secondary`), sin movimiento apresurado.
- `overridden`: usa el símbolo y el tinte elegidos independientemente de la actividad real.

## Submenú Contexto

- El menú raíz muestra una fila «Contexto» con el recuento/estado de sesiones; esta abre un submenú (`MenuSessionsInjector`).
- El encabezado del submenú muestra el número de sesiones activas durante las últimas 24 horas.
- Cada fila de sesión conserva su barra de tokens, antigüedad, vista previa, selector de razonamiento/detalle, y acciones de restablecimiento, compactación y eliminación.
- Los mensajes de carga, desconexión y error al cargar sesiones se muestran dentro del submenú Contexto.
- Las secciones de uso y costes permanecen en el nivel raíz debajo de Contexto para que puedan consultarse de un vistazo sin abrir el submenú.

## Texto de la fila de estado (menú)

- Mientras hay trabajo activo: `<Session role> · <activity label>` (`"\(roleLabel) · \(activity.label)"` en `MenuContentView`), donde la etiqueta del rol es `Main` o `Other`.
- Cuando está inactivo: vuelve al resumen de salud.

## Ingesta de eventos

- Fuente: eventos `agent` del canal de control, enrutados por `ControlChannel.routeWorkActivity(from:)`.
- Campos analizados:
  - `stream: "job"` con `data.state` para iniciar/detener.
  - `stream: "tool"` con `data.phase`, `data.name`, `data.meta`/`data.args` opcionales.
- Las etiquetas de herramientas proceden de `ToolDisplayRegistry.resolve(name:args:meta:)`; los nombres no resueltos usan como alternativa el nombre sin procesar de la herramienta.

## Anulación de depuración

- Settings > Debug > selector "Icon override":
  - `System (auto)` (predeterminado)
  - `Working: main` / `Working: other` (por tipo de herramienta: bash, lectura, escritura, edición, otro)
  - `Idle`
- Se almacena bajo la clave `openclaw.iconOverride` de `UserDefaults`; se asigna a `IconState.overridden`.

## Lista de comprobación para pruebas

- Iniciar una tarea de la sesión principal: el icono cambia inmediatamente y la fila de estado muestra la etiqueta principal.
- Iniciar una tarea de una sesión no principal mientras la principal está inactiva: el icono/estado muestra la sesión no principal y permanece estable hasta que finaliza.
- Iniciar la sesión principal mientras otra sesión está activa: el icono cambia instantáneamente a la principal.
- Ráfagas rápidas de herramientas: la insignia no parpadea (periodo de gracia de 2 s antes de borrar una herramienta finalizada, `WorkActivityStore.toolResultGrace`).
- La fila de salud reaparece cuando todas las sesiones están inactivas.

## Relacionado

- [Aplicación para macOS](/es/platforms/macos)
- [Icono de la barra de menús](/es/platforms/mac/icon)
