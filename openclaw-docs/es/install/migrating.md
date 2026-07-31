---
read_when:
    - Está trasladando OpenClaw a un nuevo portátil o servidor
    - Proviene de otro sistema de agentes y desea conservar el estado
    - Está actualizando un plugin existente en el mismo lugar
summary: 'Centro de migración: importaciones entre sistemas, traslados entre máquinas y actualizaciones de plugins'
title: Guía de migración
x-i18n:
    generated_at: "2026-07-26T05:18:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9ceb80045ab082c9cfc9e1aca59e079b6bf28b1d047265a0be40c03ebe5dac6
    source_path: install/migrating.md
    workflow: 16
---

OpenClaw admite tres rutas de migración: importar desde otro sistema de agentes, trasladar una instalación existente a una máquina nueva y actualizar un plugin en el mismo lugar.

## Importar desde otro sistema de agentes

Los proveedores de migración incluidos incorporan a OpenClaw instrucciones, servidores MCP, Skills, configuración de modelos y claves de API (previa aceptación). Los planes se previsualizan antes de realizar cualquier cambio y los secretos se ocultan en los informes. La operación independiente `openclaw migrate` está respaldada por una copia de seguridad verificada; en cambio, las importaciones durante la incorporación inicial primero preparan y verifican los artefactos locales antes de publicarlos, con la configuración confirmada antes de cualquier activación externa irreversible.

<CardGroup cols={2}>
  <Card title="Migración desde Claude" href="/es/install/migrating-claude" icon="brain">
    Importa el estado de Claude Code y Claude Desktop, incluidos `CLAUDE.md`, servidores MCP, Skills y comandos de proyecto.
  </Card>
  <Card title="Migración desde Hermes" href="/es/install/migrating-hermes" icon="feather">
    Importa la configuración, los proveedores, los servidores MCP, la memoria y las Skills de Hermes, así como las claves `.env` compatibles.
  </Card>
</CardGroup>

El punto de entrada de la CLI es [`openclaw migrate`](/es/cli/migrate). La incorporación también puede ofrecer la migración cuando detecta un origen conocido (`openclaw onboard --flow import`).

## Trasladar OpenClaw a una máquina nueva

Copie el **directorio de estado** (`~/.openclaw/` de forma predeterminada) y el **espacio de trabajo** para conservar:

- **Configuración** — `openclaw.json` y todos los ajustes del Gateway.
- **Autenticación** — `auth-profiles.json` por agente (claves de API y OAuth), además de cualquier estado de canal o proveedor ubicado en `credentials/`.
- **Sesiones** — historial de conversaciones y estado de los agentes.
- **Estado de los canales** — inicio de sesión de WhatsApp, sesión de Telegram y similares.
- **Archivos del espacio de trabajo** — `MEMORY.md`, `USER.md`, Skills y prompts.

<Tip>
Ejecute `openclaw status` en la máquina antigua para confirmar la ruta del directorio de estado. Los perfiles personalizados utilizan `~/.openclaw-<profile>/` o una ruta establecida mediante `OPENCLAW_STATE_DIR`.
</Tip>

### Pasos de migración

<Steps>
  <Step title="Detener el Gateway y crear una copia de seguridad">
    En la máquina **antigua**, detenga el Gateway para que los archivos no cambien durante la copia y, a continuación, archívelos:

    ```bash
    openclaw gateway stop
    cd ~
    tar -czf openclaw-state.tgz .openclaw
    ```

    Si utiliza varios perfiles (por ejemplo, `~/.openclaw-work`), archive cada uno por separado.

  </Step>

  <Step title="Instalar OpenClaw en la máquina nueva">
    [Instale](/es/install) la CLI (y Node si es necesario) en la máquina nueva. No hay problema si la incorporación crea un `~/.openclaw/` nuevo: se sobrescribirá en el siguiente paso.
  </Step>

  <Step title="Copiar el directorio de estado y el espacio de trabajo">
    Transfiera el archivo mediante `scp`, `rsync -a` o una unidad externa y, a continuación, extráigalo:

    ```bash
    cd ~
    tar -xzf openclaw-state.tgz
    ```

    Confirme que se incluyeron los directorios ocultos y que la propiedad de los archivos corresponde al usuario que ejecutará el Gateway.

  </Step>

  <Step title="Ejecutar Doctor y verificar">
    En la máquina nueva, ejecute [Doctor](/es/gateway/doctor) para aplicar las migraciones de configuración y reparar los servicios:

    ```bash
    openclaw doctor
    openclaw gateway restart
    openclaw status
    ```

  </Step>
</Steps>

Si Telegram o Discord utiliza la alternativa predeterminada mediante variables de entorno (`TELEGRAM_BOT_TOKEN` o `DISCORD_BOT_TOKEN`), compruebe que el archivo `.env` del directorio de estado migrado contiene esas claves sin imprimir los valores secretos:

```bash
awk -F= '/^(TELEGRAM_BOT_TOKEN|DISCORD_BOT_TOKEN)=/ { print $1 "=present" }' ~/.openclaw/.env
```

`openclaw doctor` también advierte cuando una cuenta predeterminada habilitada de Telegram o Discord no tiene ningún token configurado y la variable de entorno correspondiente no está disponible para el proceso de Doctor.

### Problemas habituales

<AccordionGroup>
  <Accordion title="El perfil o el directorio de estado no coinciden">
    Si el Gateway antiguo utilizaba `--profile` o `OPENCLAW_STATE_DIR` y el nuevo no, los canales aparecerán con la sesión cerrada y las sesiones estarán vacías. Inicie el Gateway con el **mismo** perfil o directorio de estado que migró y vuelva a ejecutar `openclaw doctor`.
  </Accordion>

  <Accordion title="Copiar únicamente openclaw.json">
    El archivo de configuración por sí solo no es suficiente. Los perfiles de autenticación de modelos se encuentran en `agents/<agentId>/agent/auth-profiles.json`, mientras que el estado de los canales y proveedores se encuentra en `credentials/`. Migre siempre el directorio de estado **completo**.
  </Accordion>

  <Accordion title="Permisos y propiedad">
    Si realizó la copia como usuario root o cambió de usuario, es posible que el Gateway no pueda leer las credenciales. Asegúrese de que el directorio de estado y el espacio de trabajo pertenezcan al usuario que ejecuta el Gateway.
  </Accordion>

  <Accordion title="Modo remoto">
    Si la interfaz de usuario apunta a un Gateway **remoto**, el host remoto es el propietario de las sesiones y el espacio de trabajo. Migre el propio host del Gateway, no el portátil local. Consulte las [preguntas frecuentes](/es/help/faq#where-things-live-on-disk).
  </Accordion>

  <Accordion title="Secretos en las copias de seguridad">
    El directorio de estado contiene perfiles de autenticación, credenciales de canales y otros estados de proveedores. Almacene las copias de seguridad cifradas, evite los canales de transferencia inseguros y rote las claves si sospecha que se han expuesto.
  </Accordion>
</AccordionGroup>

### Lista de comprobación

En la máquina nueva, confirme lo siguiente:

- [ ] `openclaw status` muestra que el Gateway está en ejecución.
- [ ] Los canales siguen conectados (no es necesario volver a vincularlos).
- [ ] El panel se abre y muestra las sesiones existentes.
- [ ] Los archivos del espacio de trabajo (memoria y configuraciones) están presentes.

## Actualizar un plugin en el mismo lugar

Las actualizaciones de plugins en el mismo lugar conservan el mismo identificador de plugin y las mismas claves de configuración, pero pueden trasladar el estado almacenado en disco a la disposición actual. Las guías de actualización específicas de cada plugin se encuentran junto a sus canales:

- [Migración de Matrix](/es/channels/matrix-migration): límites de recuperación del estado cifrado, comportamiento de las instantáneas automáticas y comandos de recuperación manual.

## Contenido relacionado

- [`openclaw migrate`](/es/cli/migrate): referencia de la CLI para importaciones entre sistemas.
- [Descripción general de la instalación](/es/install): todos los métodos de instalación.
- [Doctor](/es/gateway/doctor): comprobación del estado posterior a la migración.
- [Desinstalación](/es/install/uninstall): cómo eliminar OpenClaw de forma limpia.
