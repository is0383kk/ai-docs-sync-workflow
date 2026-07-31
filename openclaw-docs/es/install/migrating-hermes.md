---
read_when:
    - Vienes de Hermes y quieres conservar la configuración de tus modelos, tus prompts, tu memoria y tus Skills
    - Quieres saber qué importa OpenClaw automáticamente y qué permanece solo en el archivo
    - Necesita una ruta de migración limpia y automatizada mediante scripts (CI, portátil nuevo, automatización)
summary: Migra de Hermes a OpenClaw con una importación previsualizada y reversible
title: Migración desde Hermes
x-i18n:
    generated_at: "2026-07-26T05:14:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8cdb7a77cfb8ecb0504ccc322b5600c6ed671a8bf9ac866d964fdf4b3494000
    source_path: install/migrating-hermes.md
    workflow: 16
---

El proveedor de migración de Hermes incluido sigue `HERMES_HOME` y el perfil activo de Hermes, con reserva en `~/.hermes` en macOS/Linux o `%LOCALAPPDATA%\hermes` en Windows. Previsualiza todos los cambios antes de aplicarlos y oculta los secretos en los planes e informes. El comando independiente `openclaw migrate` escribe una copia de seguridad verificada; la ruta de incorporación nueva prepara la configuración, las credenciales y los archivos, y solo los publica después de que la inferencia importada se verifica. Una ruta `--from` explícita siempre tiene prioridad.

<Note>
Las importaciones requieren una configuración nueva de OpenClaw. Si ya se dispone de estado local de OpenClaw, primero deben restablecerse la configuración, las credenciales, las sesiones y el espacio de trabajo, o bien usar `openclaw migrate apply hermes` directamente con `--overwrite` después de revisar el plan.
</Note>

## Dos formas de importar

<Tabs>
  <Tab title="Asistente de incorporación">
    Detecta el directorio principal/perfil activo de Hermes y muestra una vista previa antes de aplicar los cambios.

    ```bash
    openclaw onboard --flow import
    ```

    También se puede indicar una fuente específica:

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

  </Tab>
  <Tab title="CLI">
    Usar `openclaw migrate` para ejecuciones mediante scripts o repetibles. Consultar [`openclaw migrate`](/es/cli/migrate) para obtener la referencia completa.

    ```bash
    openclaw migrate hermes --dry-run    # solo vista previa
    openclaw migrate apply hermes --yes  # aplicar sin solicitar confirmación
    ```

    Añadir `--from <path>` para anular la detección del directorio principal/perfil de Hermes.

  </Tab>
</Tabs>

## Qué se importa

<AccordionGroup>
  <Accordion title="Configuración del modelo">
    - Selección predeterminada del modelo desde `config.yaml` de Hermes.
    - Proveedores de modelos configurados y puntos de conexión personalizados de `model`, `providers` y `custom_providers`, incluidos los transportes actuales Chat Completions de Hermes, Codex Responses y Anthropic Messages.

  </Accordion>
  <Accordion title="Servidores MCP">
    Definiciones de servidores MCP de `mcp_servers` o `mcp.servers`, incluidos el estado deshabilitado, los tiempos de espera, la compatibilidad con herramientas paralelas, el ámbito de OAuth, los campos TLS compatibles y la política de herramientas nativas, de recursos y de indicaciones. Las variables de entorno y las cabeceras literales requieren consentimiento para importar credenciales. La configuración exclusiva de Hermes sobre el ciclo de vida, el muestreo, la solicitud de información, la comprobación previa, la conservación de la conexión, el paquete de CA, la clave de cliente protegida por contraseña y los clientes OAuth prerregistrados se convierte en elementos para revisión manual en lugar de en configuración no válida de OpenClaw.
  </Accordion>
  <Accordion title="Archivos del espacio de trabajo">
    - `SOUL.md` y `AGENTS.md` se copian en el espacio de trabajo del agente de OpenClaw.
    - `memories/MEMORY.md` y `memories/USER.md` se **añaden** a los archivos de memoria correspondientes de OpenClaw en lugar de sobrescribirlos.
    - Las superficies exclusivas de memoria se comportan de forma diferente: la página de memoria de incorporación y la página de importación de memoria de la interfaz de control copian estos dos archivos en `memory/imports/hermes/` para su recuperación indexada y no modifican la memoria existente del espacio de trabajo.

  </Accordion>
  <Accordion title="Configuración de memoria">
    Valores predeterminados de configuración de memoria para la memoria de archivos de OpenClaw. Los proveedores de memoria externos, como Honcho, se registran como elementos archivados o para revisión manual, de modo que puedan trasladarse de forma deliberada.
  </Accordion>
  <Accordion title="Skills">
    Las Skills que tengan un archivo `SKILL.md` en cualquier ubicación bajo `skills/` se detectan de forma recursiva, se aplanan en el directorio de Skills del espacio de trabajo de OpenClaw y se copian con sus archivos auxiliares. Se conservan los valores de configuración de cada Skill de `skills.config`.
  </Accordion>
  <Accordion title="Credenciales de autenticación">
    El comando interactivo `openclaw migrate` solicita confirmación antes de importar credenciales de autenticación, con sí seleccionado de forma predeterminada. Las importaciones aceptadas incluyen las entradas OAuth actuales de OpenAI Codex de Hermes, las entradas OAuth de OpenAI y GitHub Copilot de OpenCode, y las [claves `.env` compatibles de Hermes](/es/cli/migrate#supported-env-keys). Usar `--include-secrets` para la importación no interactiva, `--no-auth-credentials` para omitir las credenciales o la opción `--import-secrets` de incorporación. Después de importar OAuth de Hermes, no deben mantenerse Hermes y OpenClaw usando la misma concesión de actualización; es necesario volver a autenticar uno de los dos antes de ejecutar ambos.
  </Accordion>
</AccordionGroup>

## Qué permanece solo archivado

El proveedor copia lo siguiente en el directorio de informes de migración para su revisión manual, pero **no** lo carga en la configuración ni en las credenciales activas de OpenClaw:

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `plans/`, `workspace/`, `skins/` y `kanban/`
- Almacenes `pairing/` y `platforms/`, además del estado de enrutamiento/proceso del Gateway
- `state.db`, `hermes_state.db`, `projects.db`, `response_store.db`, `memory_store.db`, `verification_evidence.db`, `kanban.db` y `retaindb_queue.db`

OpenClaw se niega a ejecutar este estado o confiar en él automáticamente porque los formatos y los supuestos de confianza pueden divergir entre sistemas. Tras revisar el archivo, debe trasladarse manualmente lo que se necesite.

## Flujo recomendado

<Steps>
  <Step title="Previsualizar el plan">
    ```bash
    openclaw migrate hermes --dry-run
    ```

    El plan enumera todo lo que cambiará, incluidos los conflictos, los elementos omitidos y los elementos sensibles. Las claves anidadas que parecen contener secretos se ocultan en la salida.

  </Step>
  <Step title="Aplicar con copia de seguridad">
    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw crea y verifica una copia de seguridad antes de aplicar los cambios. Este ejemplo no interactivo solo importa estado no secreto. Ejecutar sin `--yes` para responder de forma interactiva a la solicitud de credenciales, o añadir `--include-secrets` para incluir las credenciales compatibles en una ejecución desatendida.

  </Step>
  <Step title="Ejecutar el diagnóstico">
    ```bash
    openclaw doctor
    ```

    [El diagnóstico](/es/gateway/doctor) vuelve a aplicar cualquier migración de configuración pendiente y comprueba si se introdujeron problemas durante la importación.

  </Step>
  <Step title="Reiniciar y verificar">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Confirmar que el Gateway funciona correctamente y que el modelo, la memoria y las Skills importados están cargados.

  </Step>
</Steps>

## Gestión de conflictos

La aplicación se niega a continuar cuando el plan informa de conflictos (ya existe un archivo o valor de configuración en el destino).

<Warning>
Volver a ejecutar con `--overwrite` únicamente cuando se pretenda reemplazar el destino existente. Los proveedores aún pueden escribir copias de seguridad individuales de los archivos sobrescritos en el directorio de informes de migración.
</Warning>

Los conflictos son poco habituales en una instalación nueva. Normalmente aparecen al volver a ejecutar la importación en una configuración que ya contiene modificaciones del usuario.

Si surge un conflicto durante la aplicación (por ejemplo, una condición de carrera inesperada en un archivo de configuración), ese elemento se notifica como conflicto mientras continúan los archivos, las Skills, las credenciales, los archivos y las entradas de configuración independientes. Resolver el elemento en conflicto y volver a ejecutar la importación; las importaciones de memoria idénticas son idempotentes.

## Secretos

El comando interactivo `openclaw migrate` pregunta si se desean importar las credenciales de autenticación detectadas, con sí seleccionado de forma predeterminada.

- Al aceptar, se importan las entradas OAuth actuales de OpenAI Codex de Hermes, las entradas OAuth de OpenAI y GitHub Copilot de OpenCode, y las [claves `.env` compatibles](/es/cli/migrate#supported-env-keys).
- Usar `--no-auth-credentials`, o responder no a la solicitud, para importar solo estado no secreto.
- Usar `--include-secrets` para importar credenciales en una ejecución desatendida de `--yes`.
- Usar la opción `--import-secrets` del asistente de incorporación para importar credenciales desde el asistente.

## Salida JSON para automatización

```bash
openclaw migrate hermes --dry-run --json
openclaw migrate apply hermes --json --yes
```

Con `--json` y sin `--yes`, la aplicación imprime el plan y no modifica el estado: es el modo más seguro para la CI y los scripts compartidos.

## Solución de problemas

<AccordionGroup>
  <Accordion title="La aplicación se niega a continuar debido a conflictos">
    Inspeccionar la salida del plan. Cada conflicto identifica la ruta de origen y el destino existente. Decidir para cada elemento si se omite, se edita el destino o se vuelve a ejecutar con `--overwrite`.
  </Accordion>
  <Accordion title="Hermes se encuentra fuera de ~/.hermes">
    Pasar `--from /actual/path` (CLI) o `--import-source /actual/path` (incorporación).
  </Accordion>
  <Accordion title="La incorporación se niega a importar en una configuración existente">
    Las importaciones de incorporación requieren una configuración nueva. Se puede restablecer el estado y volver a realizar la incorporación, o usar `openclaw migrate apply hermes` directamente, que admite `--overwrite` y el control explícito de las copias de seguridad.
  </Accordion>
  <Accordion title="Las claves de API no se importaron">
    El comando interactivo `openclaw migrate` solo importa las claves de API cuando se acepta la solicitud de credenciales. Las ejecuciones no interactivas de `--yes` requieren `--include-secrets`; las importaciones de incorporación requieren `--import-secrets`. Solo se reconocen las [claves `.env` compatibles](/es/cli/migrate#supported-env-keys); las demás variables `.env` se ignoran.
  </Accordion>
</AccordionGroup>

## Recursos relacionados

- [`openclaw migrate`](/es/cli/migrate): referencia completa de la CLI, contrato del Plugin y estructuras JSON.
- [Incorporación](/es/cli/onboard): flujo del asistente y opciones no interactivas.
- [Migración](/es/install/migrating): trasladar una instalación de OpenClaw entre equipos.
- [Diagnóstico](/es/gateway/doctor): comprobación del estado posterior a la migración.
- [Espacio de trabajo del agente](/es/concepts/agent-workspace): ubicación de `SOUL.md`, `AGENTS.md` y los archivos de memoria.
