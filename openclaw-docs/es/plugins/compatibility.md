---
read_when:
    - Mantienes un plugin de OpenClaw
    - Aparece una advertencia de compatibilidad del plugin
    - Está planificando una migración del SDK de plugins o del manifiesto
summary: Contratos de compatibilidad de Plugin, metadatos de obsolescencia y expectativas de migración
title: Compatibilidad del Plugin
x-i18n:
    generated_at: "2026-07-26T05:20:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cf1dfce9e0538e78138ff80a6807ee36267a07d3eee6f19bd8e56e5c0c9cd3
    source_path: plugins/compatibility.md
    workflow: 16
---

OpenClaw mantiene conectados los contratos de Plugin anteriores mediante adaptadores de
compatibilidad con nombre antes de eliminarlos. Esto protege los plugins incluidos y externos
existentes mientras evolucionan los contratos del SDK, el manifiesto, la configuración inicial,
la configuración y el entorno de ejecución del agente.

## Registro de compatibilidad

Los contratos de compatibilidad de Plugin se registran en el registro central en
`src/plugins/compat/registry.ts`. Cada registro contiene:

- un código de compatibilidad estable
- estado: `active`, `deprecated`, `removal-pending` o `removed`
- propietario: `sdk`, `config`, `setup`, `channel`, `provider`, `plugin-execution`,
  `agent-runtime` o `core`
- fechas de introducción y obsolescencia, cuando corresponda
- una fecha de eliminación exacta una vez que el mantenedor responsable la apruebe; si se omite
  `removeAfter`, una superficie obsoleta no podrá eliminarse
- orientación sobre la sustitución
- documentación, diagnósticos y pruebas que cubren el comportamiento anterior y el nuevo

El registro es la fuente para la planificación de los mantenedores y las futuras
comprobaciones del inspector de plugins. Si cambia un comportamiento orientado a plugins,
añada o actualice el registro de compatibilidad en el mismo cambio que añade el adaptador.

La compatibilidad de las reparaciones y migraciones de Doctor se registra por separado en
`src/commands/doctor/shared/deprecation-compat.ts`. Esos registros cubren las formas de
configuración antiguas, las disposiciones del registro de instalaciones y las capas de compatibilidad de reparación que quizá deban
seguir disponibles después de eliminar la ruta de compatibilidad del entorno de ejecución.

Las revisiones de versiones deben comprobar ambos registros. No elimine una migración de
Doctor solo porque haya caducado el registro correspondiente de compatibilidad del entorno de ejecución o de la configuración;
verifique primero que no exista ninguna ruta de actualización compatible que aún necesite
la reparación. Vuelva a validar también cada anotación de sustitución durante la planificación de la versión,
ya que la propiedad de los plugins y el alcance de la configuración pueden cambiar a medida que los proveedores
y los canales salen del núcleo.

## Política de obsolescencia

OpenClaw no debe eliminar un contrato de Plugin documentado en la misma versión
que introduce su sustituto. Secuencia de migración:

1. Añada el nuevo contrato.
2. Mantenga conectado el comportamiento anterior mediante un adaptador de compatibilidad con nombre.
3. Emita diagnósticos o advertencias cuando los autores de plugins puedan actuar.
4. Documente la sustitución y el calendario.
5. Pruebe tanto las rutas anteriores como las nuevas.
6. Espere durante el periodo de migración anunciado.
7. Elimine únicamente con una aprobación explícita para una versión con cambios incompatibles.

Los registros obsoletos deben incluir una fecha de inicio de la advertencia, el sustituto, un enlace a la
documentación y una fecha final de eliminación que no sea posterior a tres meses desde el inicio de la advertencia.
No añada una ruta de compatibilidad obsoleta con un periodo de
eliminación indefinido, salvo que los mantenedores decidan explícitamente que es una compatibilidad
permanente y la marquen como `active`.

## Áreas de compatibilidad actuales

La revisión de julio de 2026 eliminó los alias caducados del SDK raíz, el manifiesto, el proveedor, el entorno de ejecución,
la marca del registro y la configuración web propiedad de plugins. Las migraciones de Doctor siguen
registrándose por separado para que las rutas de actualización compatibles aún puedan reparar configuraciones antiguas.

Las áreas de compatibilidad con fecha restantes son:

- los periodos de las subrutas del SDK de agosto y septiembre que se indican en la guía de migración
- los alias de enlaces `api.on("deactivate", ...)` y `api.on("subagent_spawning", ...)`
- el registro de incrustaciones específico de memoria y el puente del almacén de sesiones beta.5
- los alias de devoluciones de llamada entrantes de WhatsApp que se describen a continuación
- el análisis explícito del destino del canal y `openclaw/plugin-sdk/messaging-targets`
- los alias del agente Pi integrado
- los alias distribuidos del SDK del arnés de agentes, cuya eliminación está pendiente de una nueva
  decisión de migración documentada externamente

Los registros activos sin fecha cubren comportamientos compatibles, no deuda de eliminación,
incluidas las indicaciones de activación, la captura de plugins, la habilitación de plugins incluidos
y la reserva de configuración de canales generada.

### Alias planos de devoluciones de llamada entrantes de WhatsApp

Las devoluciones de llamada del entorno de ejecución de WhatsApp entregan `WebInboundMessage`: los contextos
anidados canónicos `event`, `payload`, `quote`, `group` y `platform`, además de
alias planos obsoletos para los campos de devolución de llamada distribuidos. El código nuevo de devoluciones de llamada
debe leer los contextos anidados. El código que construya mensajes de devolución de llamada anidados
limpios puede usar `WebInboundCallbackMessage`; los receptores de compatibilidad que
aún introduzcan mensajes antiguos planos de pruebas o plugins deben usar
`LegacyFlatWebInboundMessage` o `WebInboundMessageInput`.

Los alias planos seguirán disponibles hasta el **2026-08-30**; este periodo se aplica
solo al acceso mediante alias planos, no a la forma anidada, que es el contrato canónico
del entorno de ejecución. La anotación `@deprecated` de TypeScript de cada alias plano
indica su sustituto anidado exacto. Ejemplos habituales:

- `id`, `timestamp` y `isBatched` pasan a `event`.
- `body`, `mediaPath`, `mediaType`, `mediaFileName`, `mediaUrl`, `location`
  y `untrustedStructuredContext` pasan a `payload`.
- `to`, `chatId`, los campos del remitente y del propio usuario, `sendComposing`, `reply(...)` y
  `sendMedia(...)` pasan a `platform`.
- Los campos `replyTo*` pasan a `quote`; los campos de asunto, participante y mención
  del grupo pasan a `group`.

`payload.untrustedStructuredContext` se extrae de las cargas útiles entrantes del
proveedor. Los plugins deben examinar `label`, `source` y `type` antes de
considerar que su `payload` es autoritativo.

### Campos de admisión entrantes de WhatsApp

Los mensajes de devolución de llamada de WhatsApp aceptados contienen `admission`, un sobre
seguro para exposición pública que representa la decisión de control de acceso que admitió el mensaje. El código nuevo
de devoluciones de llamada debe leer los datos de admisión desde `msg.admission` en lugar de
los campos de admisión de nivel superior anteriores.

Los campos de nivel superior seguirán disponibles hasta el **2026-08-30**. La anotación
`@deprecated` de TypeScript de cada campo indica su sustituto:

- `from` y `conversationId` pasan a `admission.conversation.id`.
- `accountId` pasa a `admission.accountId`.
- `accessControlPassed` es una vista de compatibilidad derivada de
  `admission.ingress.decision === "allow"`; en los mensajes que ya contienen
  `admission`, escribir el booleano heredado no reescribe el grafo
  de entrada.
- `chatType` pasa a `admission.conversation.kind`.

## Paquete del inspector de plugins

El inspector de plugins debe residir fuera del repositorio central de OpenClaw como un
paquete o repositorio independiente respaldado por los contratos versionados de compatibilidad y
manifiesto. La CLI inicial debe ser:

```sh
openclaw-plugin-inspector ./my-plugin
```

Debe emitir la validación del manifiesto y el esquema, la versión de compatibilidad
del contrato que se comprueba, las comprobaciones de metadatos de instalación y origen, las comprobaciones de importaciones
de rutas frías y las advertencias de obsolescencia y compatibilidad. Use `--json` para obtener una salida estable
legible por máquinas en las anotaciones de CI. El núcleo de OpenClaw debe exponer
los contratos y accesorios de prueba que el inspector pueda consumir, pero no debe publicar el
binario del inspector desde el paquete principal `openclaw`.

### Canal de aceptación para mantenedores

Use Blacksmith Testbox respaldado por Crabbox para el canal de aceptación de paquetes
instalables cuando valide el inspector externo con paquetes de Plugin de
OpenClaw. Ejecútelo desde un repositorio de OpenClaw limpio después de compilar el paquete:

```sh
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "pnpm install && pnpm build && npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/telegram --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/discord --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- <clawhub-plugin-dir> --json"
```

Mantenga este canal opcional para los mantenedores, ya que instala un paquete npm
externo y puede inspeccionar paquetes de Plugin clonados fuera del repositorio. Las protecciones del
repositorio local cubren el mapa de exportaciones del SDK, los metadatos del registro de compatibilidad,
la reducción de importaciones obsoletas del SDK y los límites de importación de extensiones incluidas;
la prueba del inspector en Testbox cubre el paquete tal como lo consumen los autores de plugins
externos.

## Notas de la versión

Las notas de la versión deben incluir las próximas obsolescencias de plugins con las fechas previstas
y enlaces a la documentación de migración, antes de que una ruta de compatibilidad pase a
`removal-pending` o `removed`.
