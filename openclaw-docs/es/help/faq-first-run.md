---
read_when:
    - Instalación nueva, incorporación bloqueada o errores en la primera ejecución
    - Elección de la autenticación y las suscripciones a proveedores
    - No se puede acceder a docs.openclaw.ai, no se puede abrir el panel, la instalación está bloqueada
sidebarTitle: First-run FAQ
summary: 'Preguntas frecuentes: inicio rápido y configuración de la primera ejecución — instalación, incorporación, autenticación, suscripciones y errores iniciales'
title: 'Preguntas frecuentes: configuración inicial'
x-i18n:
    generated_at: "2026-07-26T05:12:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e1c93b89da625ae5f092db854c9b74adc005be75dd913af4bf89ed1a4f35396a
    source_path: help/faq-first-run.md
    workflow: 16
---

Inicio rápido y preguntas y respuestas sobre la primera ejecución. Para las operaciones cotidianas, los modelos, la autenticación, las sesiones
y la solución de problemas, consulte las [preguntas frecuentes](/es/help/faq) principales.

## Inicio rápido y configuración inicial

<AccordionGroup>
  <Accordion title="Estoy bloqueado; la forma más rápida de desbloquearme">
    Use un agente de IA local que pueda **ver su máquina**. La mayoría de los casos de «estoy bloqueado» son
    **problemas locales de configuración o del entorno** que un asistente remoto no puede inspeccionar, por lo que esta opción es mejor que
    preguntar en Discord.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Proporcione al agente el código fuente completo mediante la instalación modificable (git), para que pueda leer
    el código y la documentación, y razonar sobre la versión exacta que se está ejecutando:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Pida al agente que planifique y supervise la corrección paso a paso y que, después, ejecute únicamente los
    comandos necesarios; las diferencias más pequeñas son más fáciles de auditar.

    Comparta estos resultados cuando solicite ayuda (en Discord o en una incidencia de GitHub):

    | Comando | Muestra |
    | --- | --- |
    | `openclaw status` | Estado del Gateway/agente e instantánea básica de la configuración |
    | `openclaw status --all` | Diagnóstico completo de solo lectura, listo para pegar |
    | `openclaw models status` | Autenticación del proveedor y disponibilidad de modelos |
    | `openclaw doctor` | Valida y repara problemas comunes de configuración y estado |
    | `openclaw logs --follow` | Seguimiento del registro en tiempo real |
    | `openclaw gateway status --deep` | Comprobación exhaustiva del estado del Gateway, la configuración y los plugins |
    | `openclaw health --verbose` | Informe de estado detallado |

    ¿Ha encontrado un error real o una corrección? Abra una incidencia o envíe un PR:
    [Incidencias](https://github.com/openclaw/openclaw/issues) /
    [Pull requests](https://github.com/openclaw/openclaw/pulls).

    Ciclo rápido de depuración: [Primeros 60 segundos si algo no funciona](/es/help/faq#first-60-seconds-if-something-is-broken).
    Documentación de instalación: [Instalación](/es/install), [Opciones del instalador](/es/install/installer), [Actualización](/es/install/updating).

  </Accordion>

  <Accordion title="Heartbeat sigue omitiéndose. ¿Qué significan los motivos de omisión?">
    | Motivo de omisión | Significado |
    | --- | --- |
    | `quiet-hours` | Fuera del intervalo de horas activas configurado |
    | `empty-heartbeat-file` | Existe el borrador del monitor de Heartbeat, pero solo contiene una estructura vacía, de comentarios, encabezados, delimitadores o listas de comprobación vacías |
    | `alerts-disabled` | Toda la visibilidad de Heartbeat está desactivada (`showOk`, `showAlerts` y `useIndicator` están deshabilitados) |

    Los bloques antiguos de Heartbeat `tasks:` se migran a trabajos de Cron programados de forma independiente con `openclaw doctor --fix`.

    Documentación: [Heartbeat](/es/gateway/heartbeat), [Automatización](/es/automation).

  </Accordion>

  <Accordion title="Forma recomendada de instalar y configurar OpenClaw">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    Desde el código fuente (colaboradores/desarrollo):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build
    openclaw onboard
    ```

    ¿Todavía no hay una instalación global? Ejecute `pnpm openclaw onboard` en su lugar. Si faltan los recursos de la interfaz de control,
    la incorporación intenta compilarlos automáticamente y, si falla, recurre a `pnpm ui:build`.

  </Accordion>

  <Accordion title="¿Cómo abro el panel después de la incorporación?">
    La incorporación abre el navegador con una URL limpia del panel (sin token) inmediatamente después de
    la configuración e imprime el enlace en el resumen. Mantenga esa pestaña abierta; si no se abrió,
    copie y pegue la URL impresa en la misma máquina.
  </Accordion>

  <Accordion title="¿Cómo autentico el panel en localhost y de forma remota?">
    **Localhost (misma máquina):**

    - Abra `http://127.0.0.1:18789/`.
    - Si solicita autenticación mediante secreto compartido, pegue el token o la contraseña configurados en los ajustes de la interfaz de control.
    - Origen del token: `gateway.auth.token` (o `OPENCLAW_GATEWAY_TOKEN`).
    - Origen de la contraseña: `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`).
    - ¿Aún no se ha configurado ningún secreto compartido? Ejecute `openclaw doctor --generate-gateway-token` (o `openclaw doctor --fix --generate-gateway-token`).

    **Fuera de localhost:**

    - **Tailscale Serve** (recomendado): mantenga el enlace en loopback, ejecute `openclaw gateway --tailscale serve` y abra `https://<magicdns>/`. Con `gateway.auth.allowTailscale: true`, los encabezados de identidad satisfacen la autenticación de la interfaz de control/WebSocket (no es necesario pegar un secreto compartido; se presupone que el host del Gateway es de confianza); las API HTTP siguen necesitando autenticación mediante secreto compartido, salvo que se utilice deliberadamente la entrada privada `none` o la autenticación HTTP mediante proxy de confianza.
      Los intentos simultáneos de Serve con autenticación incorrecta procedentes del mismo cliente se serializan antes de que el limitador de autenticaciones fallidas los registre, por lo que un segundo reintento incorrecto ya puede mostrar `retry later`.
    - **Enlace a Tailnet**: ejecute `openclaw gateway --bind tailnet --token "<token>"` (o configure la autenticación mediante contraseña), abra `http://<tailscale-ip>:18789/` y pegue el secreto compartido correspondiente en los ajustes del panel.
    - **Proxy inverso con reconocimiento de identidad**: mantenga el Gateway detrás de un proxy de confianza, establezca `gateway.auth.mode: "trusted-proxy"` y abra la URL del proxy. Los proxies loopback del mismo host necesitan `gateway.auth.trustedProxy.allowLoopback: true` explícito.
    - **Túnel SSH**: `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host` y, después, abra `http://127.0.0.1:18789/`. La autenticación mediante secreto compartido sigue aplicándose a través del túnel; pegue el token o la contraseña configurados si se solicitan.

    Consulte [Panel](/es/web/dashboard) y [Superficies web](/es/web) para obtener información sobre los modos de enlace y la autenticación.

  </Accordion>

  <Accordion title="¿Por qué hay dos configuraciones de aprobación de ejecución para las aprobaciones por chat?">
    Controlan capas diferentes:

    - `approvals.exec`: reenvía las solicitudes de aprobación a destinos de chat.
    - `channels.<channel>.execApprovals`: convierte ese canal en un cliente de aprobación nativo para las aprobaciones de ejecución.

    La política de ejecución del host sigue siendo la verdadera puerta de aprobación; la configuración del chat solo controla dónde
    aparecen las solicitudes y cómo responden las personas.

    Rara vez se necesitan ambas:

    - Si el chat ya admite comandos y respuestas, `/approve` en el mismo chat funciona mediante la ruta compartida.
    - Cuando un canal nativo compatible puede deducir de forma segura quién puede aprobar, OpenClaw activa automáticamente las aprobaciones nativas que priorizan los mensajes directos si `channels.<channel>.execApprovals.enabled` no está establecido o es `"auto"`.
    - Cuando hay disponibles tarjetas o botones de aprobación nativos, esa interfaz es la principal; mencione un comando manual `/approve` únicamente si el resultado de la herramienta indica que las aprobaciones por chat no están disponibles.
    - Use `approvals.exec` únicamente cuando las solicitudes también deban llegar a otros chats o salas de operaciones explícitas.
    - Use `channels.<channel>.execApprovals.target: "channel"` o `"both"` únicamente cuando se desee volver a publicar las solicitudes de aprobación en la sala o el tema de origen.
    - Las aprobaciones de plugins son independientes: de forma predeterminada, `/approve` en el mismo chat; reenvío opcional mediante `approvals.plugin`; y solo algunos canales nativos mantienen también el procesamiento nativo para ellas.

    En resumen: el reenvío sirve para el enrutamiento; la configuración del cliente nativo proporciona una experiencia de usuario más completa y específica del canal.
    Consulte [Aprobaciones de ejecución](/es/tools/exec-approvals).

  </Accordion>

  <Accordion title="¿Qué entorno de ejecución necesito?">
    Se requiere Node **22.22.3+**, **24.15+** o **25.9+** (se recomienda Node 24). `pnpm` es el gestor de paquetes del repositorio.
    Bun puede instalar dependencias y ejecutar scripts de paquetes, pero no puede ejecutar la CLI ni el Gateway de OpenClaw porque carece de `node:sqlite`.
  </Accordion>

  <Accordion title="¿Se ejecuta en Raspberry Pi?">
    Sí, pero compruebe primero la RAM: Pi 5 y Pi 4 (2 GB+) son las opciones ideales; Pi 3B+ (1 GB) funciona, pero es lento; no se recomienda Pi Zero 2 W (512 MB).

    | Modelo | RAM | Idoneidad |
    | --- | --- | --- |
    | Pi 5 | 4/8 GB | Óptima |
    | Pi 4 | 4 GB | Buena |
    | Pi 4 | 2 GB | Aceptable; añada memoria de intercambio |
    | Pi 4 | 1 GB | Limitada |
    | Pi 3B+ | 1 GB | Lenta |
    | Pi Zero 2 W | 512 MB | No recomendado |

    Mínimo absoluto: 1 GB de RAM, 1 núcleo, 500 MB de espacio libre en disco y un sistema operativo de 64 bits. Como la Pi solo ejecuta
    el Gateway (los modelos llaman a API en la nube), incluso una Pi modesta soporta la carga.

    Una Pi o un VPS pequeños también pueden alojar únicamente el Gateway mientras se emparejan **nodos** en el
    portátil o teléfono para acceder localmente a la pantalla, la cámara o el lienzo, o para ejecutar comandos. Consulte [Nodos](/es/nodes).

    Guía completa de configuración: [Raspberry Pi](/es/install/raspberry-pi).

  </Accordion>

  <Accordion title="¿Algún consejo para instalar en Raspberry Pi?">
    - Use un sistema operativo de **64 bits**; no use Raspberry Pi OS de 32 bits.
    - Añada memoria de intercambio en placas de 2 GB o menos.
    - Prefiera un **SSD USB** en lugar de una tarjeta SD para mejorar el rendimiento y la vida útil.
    - Prefiera la instalación modificable (git) para poder consultar los registros y actualizar rápidamente.
    - Empiece sin canales ni Skills y añádalos uno por uno.
    - Los fallos extraños de binarios («exec format error») suelen deberse a que falta una compilación ARM64 para una herramienta opcional de una Skill.

    Guía completa: [Raspberry Pi](/es/install/raspberry-pi). Consulte también [Linux](/es/platforms/linux).

  </Accordion>

  <Accordion title="Se ha quedado bloqueado en «despierta, amigo» o la incorporación no termina de arrancar. ¿Qué hago?">
    Esa pantalla depende de que el Gateway sea accesible y esté autenticado. La TUI también envía
    «¡Despierta, amigo!» automáticamente durante el primer arranque cuando hay configurado un proveedor de modelos. Si
    se omitió la configuración del modelo o de la autenticación, la incorporación muestra una nota de «Falta la autenticación del modelo» y abre la
    TUI sin enviar nada; añada un proveedor con `openclaw configure --section model`.
    Si aparece la línea de activación **sin respuesta** y los tokens permanecen en 0, el agente nunca llegó a ejecutarse.

    1. Reinicie el Gateway:

    ```bash
    openclaw gateway restart
    ```

    2. Compruebe el estado y la autenticación:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. ¿Sigue bloqueado? Ejecute:

    ```bash
    openclaw doctor
    ```

    Si el Gateway es remoto, confirme que la conexión por túnel/Tailscale esté activa y que la interfaz
    apunte al Gateway correcto. Consulte [Acceso remoto](/es/gateway/remote).

  </Accordion>

  <Accordion title="¿Puedo migrar mi configuración a una máquina nueva sin repetir la incorporación?">
    Sí. Copie el **directorio de estado** y el **espacio de trabajo** y, después, ejecute Doctor una vez:

    1. Instale OpenClaw en la máquina nueva.
    2. Copie `$OPENCLAW_STATE_DIR` (valor predeterminado: `~/.openclaw`) desde la máquina anterior.
    3. Copie el espacio de trabajo (valor predeterminado: `~/.openclaw/workspace`).
    4. Ejecute `openclaw doctor` y reinicie el servicio del Gateway.

    Esto conserva la configuración, los perfiles de autenticación, las credenciales de WhatsApp, las sesiones y la memoria; mantiene
    el bot exactamente igual, siempre que se copien **ambas** ubicaciones. En modo remoto, el
    host del Gateway es propietario del almacén de sesiones y del espacio de trabajo.

    **Importante:** si solo confirma y envía el espacio de trabajo a GitHub, se crea una copia de seguridad de
    **la memoria y los archivos de arranque**, pero no del historial de sesiones ni de la autenticación. Estos se encuentran en
    `~/.openclaw/` (por ejemplo, `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`).

    Temas relacionados: [Migración](/es/install/migrating), [Ubicación de los elementos en el disco](/es/help/faq#where-things-live-on-disk),
    [Espacio de trabajo del agente](/es/concepts/agent-workspace), [Doctor](/es/gateway/doctor),
    [Modo remoto](/es/gateway/remote).

  </Accordion>

  <Accordion title="¿Dónde puedo consultar las novedades de la versión más reciente?">
    Consulte el registro de cambios de GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Las entradas más recientes aparecen en la parte superior. Si la sección superior es **Sin publicar**, la siguiente sección
    con fecha corresponde a la última versión publicada. Las entradas se agrupan en **Aspectos destacados**, **Cambios**
    y **Correcciones** (además de secciones de documentación u otras cuando sea necesario).

  </Accordion>

  <Accordion title="No se puede acceder a docs.openclaw.ai (error de SSL)">
    Algunas conexiones de Comcast/Xfinity bloquean incorrectamente `docs.openclaw.ai` mediante Xfinity
    Advanced Security. Desactívelo o añada `docs.openclaw.ai` a la lista de permitidos y vuelva a intentarlo. Ayúdenos
    a conseguir que se desbloquee: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    ¿Sigues bloqueado? La documentación está replicada en GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Diferencia entre estable y beta">
    **Estable** y **beta** son **dist-tags de npm**, no líneas de código independientes:

    - `latest` = estable
    - `beta` = compilación preliminar para pruebas (recurre a `latest` cuando beta no existe o es anterior a la versión estable actual)

    Una versión estable suele publicarse primero en **beta** y, después, un paso explícito de promoción
    mueve esa misma versión a `latest` sin cambiar el número de versión. Los mantenedores
    también pueden publicar directamente en `latest`. Por eso beta y estable pueden apuntar a la
    **misma versión** después de la promoción.

    Consulta qué ha cambiado: [CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md).

    Para consultar comandos de instalación de una sola línea y la diferencia entre beta y dev, consulta el siguiente acordeón.

  </Accordion>

  <Accordion title="¿Cómo instalo la versión beta y cuál es la diferencia entre beta y dev?">
    **Beta** es el dist-tag de npm `beta` (puede coincidir con `latest` después de la promoción).
    **Dev** es la punta móvil de `main` (git); cuando se publica en npm, usa el dist-tag `dev`.

    Comandos de una sola línea (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Instalador de Windows (PowerShell): `iwr -useb https://openclaw.ai/install.ps1 | iex`

    Más detalles: [Canales de desarrollo](/es/install/development-channels) y [Opciones del instalador](/es/install/installer).

  </Accordion>

  <Accordion title="¿Cómo pruebo las novedades más recientes?">
    Hay dos opciones:

    1. **Canal dev (instalación existente):**

    ```bash
    openclaw update --channel dev
    ```

    Esto cambia a un checkout de git de `main`, aplica rebase sobre el repositorio upstream, compila e instala
    la CLI desde ese checkout.

    2. **Instalación modificable (git) (equipo nuevo):**

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Es preferible clonar manualmente:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Documentación: [Actualización](/es/cli/update), [Canales de desarrollo](/es/install/development-channels), [Instalación](/es/install).

  </Accordion>

  <Accordion title="¿Cuánto suelen tardar la instalación y la incorporación?">
    Guía aproximada:

    - **Instalación:** 2-5 minutos.
    - **Incorporación con QuickStart:** unos minutos (Gateway en loopback, token automático, espacio de trabajo predeterminado).
    - **Incorporación avanzada/completa:** tarda más cuando el inicio de sesión del proveedor, la vinculación de canales, la instalación del daemon, las descargas de red o las Skills requieren configuración adicional.

    El asistente muestra estos plazos desde el principio. Omite los pasos opcionales y vuelve más tarde con
    `openclaw configure`.

    ¿Se ha quedado bloqueado? Consulta [Estoy bloqueado](#quick-start-and-first-run-setup) más arriba.

  </Accordion>

  <Accordion title="¿El instalador está bloqueado? ¿Cómo obtengo más información?">
    Vuelve a ejecutarlo con `--verbose`:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    `install.ps1` no tiene una opción específica para salida detallada; ejecútalo mediante `Set-PSDebug -Trace 1` /
    `-Trace 0`. Referencia completa de opciones: [Opciones del instalador](/es/install/installer).

  </Accordion>

  <Accordion title="La instalación en Windows indica que no se encontró git o que no se reconoce openclaw">
    Dos problemas comunes en Windows:

    **1) Error de npm spawn git / no se encontró git**

    - Instala **Git for Windows** y asegúrate de que `git` esté en PATH.
    - Cierra y vuelve a abrir PowerShell; después, ejecuta de nuevo el instalador.

    **2) openclaw no se reconoce después de la instalación**

    - La carpeta global de ejecutables de npm no está en PATH.
    - Compruébala: `npm config get prefix`.
    - Añade ese directorio al PATH del usuario (no se necesita el sufijo `\bin`; en la mayoría de los sistemas es `%AppData%\npm`).
    - Cierra y vuelve a abrir PowerShell.

    ¿Prefieres una aplicación de escritorio? Usa **Windows Hub**. Para una configuración solo mediante terminal, se admiten
    tanto el instalador de PowerShell como las rutas de Gateway para WSL2. Documentación: [Windows](/es/platforms/windows).

  </Accordion>

  <Accordion title="La salida de exec en Windows muestra texto chino ilegible: ¿qué debo hacer?">
    Por lo general, se debe a una discrepancia de página de códigos de la consola en los shells nativos de Windows.

    Síntomas: la salida de `system.run`/`exec` muestra el texto chino con caracteres ilegibles; el mismo comando
    se ve correctamente en otro perfil de terminal.

    Solución temporal en PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Después, reinicia el Gateway y vuelve a intentarlo:

    ```powershell
    openclaw gateway restart
    ```

    ¿Sigue ocurriendo en la versión más reciente de OpenClaw? Consulta o notifícalo aquí: [Incidencia n.º 30640](https://github.com/openclaw/openclaw/issues/30640).

  </Accordion>

  <Accordion title="La documentación no respondió a mi pregunta: ¿cómo obtengo una respuesta mejor?">
    Usa la instalación modificable (git) para disponer localmente del código fuente y la documentación completos; después, pregunta
    al bot (o a Claude/Codex) **desde esa carpeta** para que pueda leer el repositorio y responder con precisión.

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Más detalles: [Instalación](/es/install) y [Opciones del instalador](/es/install/installer).

  </Accordion>

  <Accordion title="¿Cómo instalo OpenClaw en Linux?">
    - Ruta rápida para Linux e instalación del servicio: [Linux](/es/platforms/linux).
    - Guía completa: [Primeros pasos](/es/start/getting-started).
    - Instalador y actualizaciones: [Instalación y actualizaciones](/es/install/updating).

  </Accordion>

  <Accordion title="¿Cómo instalo OpenClaw en un VPS?">
    Cualquier VPS con Linux sirve. Instálalo en el servidor y accede al Gateway mediante SSH/Tailscale.

    Guías: [exe.dev](/es/install/exe-dev), [Hetzner](/es/install/hetzner), [Fly.io](/es/install/fly).
    Acceso remoto: [Gateway remoto](/es/gateway/remote).

  </Accordion>

  <Accordion title="¿Dónde están las guías de instalación en la nube o en VPS?">
    Centro de alojamiento con proveedores habituales:

    - [Alojamiento en VPS](/es/vps) (todos los proveedores en un solo lugar)
    - [Fly.io](/es/install/fly)
    - [Hetzner](/es/install/hetzner)
    - [exe.dev](/es/install/exe-dev)

    En la nube, el **Gateway se ejecuta en el servidor** y se accede a él desde un portátil o teléfono
    mediante la interfaz de control (o Tailscale/SSH). El estado y el espacio de trabajo residen en el servidor, por lo que
    debe tratarse el host como fuente de verdad y realizar copias de seguridad.

    Vincula **nodos** (Mac/iOS/Android/sin interfaz) con ese Gateway en la nube para usar localmente
    la pantalla, la cámara o el lienzo, o ejecutar comandos en el portátil mientras el Gateway permanece en
    la nube.

    Centro: [Plataformas](/es/platforms). Acceso remoto: [Gateway remoto](/es/gateway/remote).
    Nodos: [Nodos](/es/nodes), [CLI de nodos](/es/cli/nodes).

  </Accordion>

  <Accordion title="¿Puedo pedirle a OpenClaw que se actualice a sí mismo?">
    Es posible, pero no se recomienda. El proceso de actualización puede reiniciar el Gateway (interrumpiendo la
    sesión activa), puede requerir un checkout de git limpio y puede solicitar confirmación.
    Es más seguro ejecutar las actualizaciones desde un shell como operador.

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|extended-stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Automatización desde un agente:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Documentación: [Actualización](/es/cli/update), [Cómo actualizar](/es/install/updating).

  </Accordion>

  <Accordion title="¿Qué hace exactamente la incorporación?">
    `openclaw onboard` es la ruta de configuración recomendada. En **modo local**, guía por los siguientes pasos:

    1. **Modelo/autenticación**: OAuth del proveedor, claves de API o autenticación manual (incluidas opciones locales como LM Studio); selección de un modelo predeterminado.
    2. **Espacio de trabajo**: ubicación y archivos de arranque.
    3. **Gateway**: puerto, dirección de enlace, modo de autenticación y exposición mediante Tailscale.
    4. **Canales**: canales de chat integrados y de plugins oficiales: iMessage, Discord, Feishu, Google Chat, Mattermost, Microsoft Teams, QQ Bot, Signal, Slack, Telegram, WhatsApp y más.
    5. **Daemon**: LaunchAgent (macOS), unidad de usuario de systemd (Linux/WSL2) o tarea programada nativa de Windows.
    6. **Comprobación de estado**: inicia el Gateway y verifica que esté ejecutándose.
    7. **Skills**: instala las skills recomendadas y las dependencias opcionales.

    Indica desde el principio la duración prevista y avisa si el modelo configurado es desconocido
    o no tiene autenticación. Desglose completo: [Incorporación (CLI)](/es/start/wizard).

  </Accordion>

  <Accordion title="¿Necesito una suscripción a Claude u OpenAI para ejecutar esto?">
    No. Ejecuta OpenClaw con **claves de API** (Anthropic/OpenAI/otros) o **modelos exclusivamente locales**
    para que los datos permanezcan en el dispositivo. Las suscripciones (Claude Pro/Max, ChatGPT/Codex) son
    formas opcionales de autenticarse con esos proveedores.

    Para Anthropic: una **clave de API** ofrece la facturación estándar por uso; **Claude CLI**
    reutiliza un inicio de sesión existente de Claude Code en el mismo host. Anthropic considera actualmente
    la ruta no interactiva `claude -p` de Claude CLI como uso programático o del Agent SDK que
    sigue consumiendo los límites del plan de la suscripción; consulta la documentación actual de facturación de Anthropic
    antes de confiar en el comportamiento de la suscripción. Para hosts de Gateway de larga duración y
    automatización compartida, una clave de API de Anthropic es la opción más predecible.

    OAuth de OpenAI Codex (suscripción a ChatGPT/Codex) es totalmente compatible con los modelos de agente.
    OpenClaw también admite opciones alojadas de tipo suscripción, como **Qwen Cloud
    Coding Plan**, **MiniMax Coding Plan** y **Z.AI / GLM Coding Plan**.

    Documentación: [Anthropic](/es/providers/anthropic), [OpenAI](/es/providers/openai),
    [Qwen Cloud](/es/providers/qwen), [MiniMax](/es/providers/minimax), [Z.AI (GLM)](/es/providers/zai),
    [Modelos locales](/es/gateway/local-models), [Modelos](/es/concepts/models).

  </Accordion>

  <Accordion title="¿Puedo usar una suscripción a Claude Max sin una clave de API?">
    Sí. OpenClaw permite reutilizar Claude CLI con los planes Pro/Max/Team/Enterprise. Anthropic
    considera actualmente la ruta `claude -p` que usa OpenClaw como uso del plan de suscripción sujeto
    a sus límites, no como una asignación gratuita independiente; consulta
    [Anthropic](/es/providers/anthropic) para conocer los detalles actuales de facturación y acceder a los enlaces de
    los artículos de soporte de Anthropic. Para obtener la configuración del lado del servidor más predecible, usa en su lugar una
    clave de API de Anthropic.
  </Accordion>

  <Accordion title="¿Se admite la autenticación mediante suscripción a Claude (Claude Pro o Max)?">
    Sí, mediante la reutilización de Claude CLI. El tratamiento de facturación de Anthropic para el uso de `claude -p`/Agent SDK
    ha cambiado con el tiempo; consulta [Anthropic](/es/providers/anthropic) para conocer el estado actual y
    acceder a enlaces con fecha a los artículos de soporte de Anthropic antes de confiar en un comportamiento
    de facturación específico.

    La autenticación mediante token de configuración de Anthropic también sigue siendo una ruta de token compatible, pero OpenClaw prefiere
    reutilizar la CLI de Claude y `claude -p` cuando estén disponibles. Para cargas de trabajo de producción o multiusuario,
    una clave de API de Anthropic sigue siendo la opción más segura y predecible. Otras
    opciones alojadas con modalidad de suscripción: [OpenAI](/es/providers/openai), [Qwen Cloud](/es/providers/qwen),
    [MiniMax](/es/providers/minimax), [Z.AI (GLM)](/es/providers/zai).

  </Accordion>

</AccordionGroup>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>

<AccordionGroup>
  <Accordion title="¿Por qué veo HTTP 429 rate_limit_error de Anthropic?">
    La **cuota o el límite de solicitudes de Anthropic** se ha agotado durante el intervalo actual. En la **CLI de Claude**,
    espere a que se restablezca el intervalo o mejore su plan. Con una **clave de API de Anthropic**,
    compruebe el uso y la facturación en Anthropic Console y aumente los límites según sea necesario.

    Si el mensaje es específicamente `Extra usage is required for long context requests`,
    la solicitud está intentando usar la ventana de contexto de 1M de Anthropic (un modelo Claude 4.x
    de 1M con disponibilidad general o la configuración heredada `params.context1m: true`), y su credencial actual no
    es apta para la facturación de contexto largo.

    Configure un **modelo alternativo** para que OpenClaw siga respondiendo mientras un proveedor tenga limitadas las solicitudes.
    Consulte [Modelos](/es/cli/models), [OAuth](/es/concepts/oauth) y
    [Anthropic 429: se requiere uso adicional para contexto largo](/es/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="¿Es compatible AWS Bedrock?">
    Sí. OpenClaw incluye un proveedor de **Amazon Bedrock (Converse)**. Cuando están presentes los marcadores
    de entorno de AWS (`AWS_ACCESS_KEY_ID`, `AWS_PROFILE`, `AWS_BEARER_TOKEN_BEDROCK`),
    OpenClaw activa automáticamente el proveedor implícito de Bedrock para descubrir modelos; de lo contrario,
    configure `plugins.entries.amazon-bedrock.config.discovery.enabled: true` o añada una entrada de
    proveedor manual. Consulte [Amazon Bedrock](/es/providers/bedrock) y [Proveedores de modelos](/es/providers/models).
    Un proxy compatible con OpenAI delante de Bedrock sigue siendo una opción válida si se prefiere un flujo administrado de claves.
  </Accordion>

  <Accordion title="¿Cómo funciona la autenticación de Codex?">
    OpenClaw admite **OpenAI Codex** mediante OAuth (inicio de sesión en ChatGPT). Una
    configuración nueva sin modelo principal usa exactamente `openai/gpt-5.6-sol` para
    la autenticación de suscripción de ChatGPT/Codex y la ejecución nativa mediante el servidor de aplicaciones de Codex.
    Al volver a autenticarse, se conserva cualquier modelo explícito existente, incluido
    `openai/gpt-5.5`. Si el espacio de trabajo de Codex no ofrece GPT-5.6, seleccione
    `openai/gpt-5.5` explícitamente; OpenClaw no cambia silenciosamente a una versión inferior. Las referencias
    de modelos con el prefijo heredado de Codex son configuraciones heredadas que repara `openclaw doctor
    --fix`. El acceso directo mediante una clave de API de OpenAI sigue disponible para las superficies de la
    API de OpenAI que no son de agentes y, mediante un perfil ordenado de clave de API `openai`, también para los modelos
    de agentes. Consulte [Proveedores de modelos](/es/concepts/model-providers) e
    [Incorporación (CLI)](/es/start/wizard).
  </Accordion>

  <Accordion title="¿Por qué OpenClaw sigue mencionando el prefijo heredado de OpenAI Codex?">
    `openai` es el identificador actual del proveedor y del perfil de autenticación tanto para las claves de API de OpenAI como para
    OAuth de ChatGPT/Codex; OpenAI Codex está integrado en él. Es posible que aún aparezca un prefijo
    heredado `openai-codex` en configuraciones antiguas y advertencias de migración:

    - `openai/gpt-5.6-sol` = configuración nueva de suscripción de ChatGPT/Codex con el entorno de ejecución nativo de Codex para los turnos del agente.
    - `openai/gpt-5.5` = selección compatible explícita para configuraciones existentes o cuentas sin acceso a GPT-5.6.
    - Referencias de modelos heredadas `openai-codex/*` = ruta heredada reparada por `openclaw doctor --fix`.
    - `openai/gpt-5.5` más un perfil ordenado de clave de API `openai` = autenticación mediante clave de API para un modelo de agente de OpenAI.
    - Identificadores heredados de perfiles de autenticación `openai-codex` = identificadores heredados migrados por `openclaw doctor --fix`.

    ¿Desea facturación directa de OpenAI Platform? Configure `OPENAI_API_KEY`. ¿Desea autenticación mediante
    suscripción de ChatGPT/Codex? Ejecute `openclaw models auth login --provider openai`. Mantenga
    las referencias de modelos bajo el proveedor canónico `openai/*`. La configuración nueva de suscripción
    usa exactamente `openai/gpt-5.6-sol`; doctor repara las referencias con prefijo heredado de Codex
    sin actualizar una selección explícita de `openai/gpt-5.5`.

  </Accordion>

  <Accordion title="¿Por qué los límites de OAuth de Codex pueden diferir de los de la web de ChatGPT?">
    OAuth de Codex usa intervalos de cuota administrados por OpenAI y dependientes del plan que pueden diferir de la
    experiencia en el sitio web o la aplicación de ChatGPT, incluso con la misma cuenta.

    `openclaw models status` muestra los intervalos de uso y cuota del proveedor visibles actualmente, pero
    no inventa ni normaliza las prestaciones de la web de ChatGPT para convertirlas en acceso directo a la API. Para la
    ruta directa de facturación y límites de OpenAI Platform, use `openai/*` con una clave de API.

  </Accordion>

  <Accordion title="¿Se admite la autenticación mediante suscripción de OpenAI (OAuth de Codex)?">
    Sí, totalmente. OpenAI permite explícitamente usar OAuth de suscripción en
    herramientas y flujos de trabajo externos como OpenClaw. La incorporación puede ejecutar el flujo de OAuth.

    Consulte [OAuth](/es/concepts/oauth), [Proveedores de modelos](/es/concepts/model-providers) e [Incorporación (CLI)](/es/start/wizard).

  </Accordion>

  <Accordion title="¿Cómo se configura OAuth de la CLI de Gemini?">
    La CLI de Gemini usa un **flujo de autenticación de Plugin**, no un identificador ni un secreto de cliente en `openclaw.json`.

    1. Instale localmente la CLI de Gemini para que `gemini` esté en `PATH`:
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. Active el Plugin: `openclaw plugins enable google`
    3. Inicie sesión: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. Modelo predeterminado tras iniciar sesión: `google/gemini-3.1-pro-preview` (entorno de ejecución `google-gemini-cli`)
    5. ¿Las solicitudes fallan después de iniciar sesión? Configure `GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` en el host del Gateway y vuelva a intentarlo.

    Los tokens de OAuth se almacenan en perfiles de autenticación en el host del Gateway. Detalles: [Google](/es/providers/google), [Proveedores de modelos](/es/concepts/model-providers).

  </Accordion>

  <Accordion title="¿Es adecuado un modelo local para conversaciones informales?">
    Por lo general, no. OpenClaw necesita un contexto amplio y una seguridad sólida; las tarjetas pequeñas truncan el contexto
    y omiten los filtros de seguridad del proveedor. Si es imprescindible, ejecute localmente la compilación de modelo **más grande**
    posible (LM Studio); consulte [Modelos locales](/es/gateway/local-models). Los modelos más pequeños o cuantizados
    aumentan el riesgo de inyección de instrucciones; consulte [Seguridad](/es/gateway/security).
  </Accordion>

  <Accordion title="¿Cómo se mantiene el tráfico de modelos alojados en una región específica?">
    Elija puntos de conexión vinculados a una región. OpenRouter ofrece opciones alojadas en EE. UU. para MiniMax, Kimi
    y GLM; elija la variante alojada en EE. UU. para mantener los datos en la región. También se pueden incluir
    Anthropic y OpenAI junto con estas opciones mediante `models.mode: "merge"`, de modo que los modelos alternativos sigan
    disponibles mientras se respeta el proveedor regional seleccionado.
  </Accordion>

  <Accordion title="¿Es necesario comprar un Mac Mini para instalarlo?">
    No. OpenClaw funciona en macOS o Linux (Windows mediante WSL2). Un Mac mini es una opción popular
    como host siempre activo, pero también sirven un VPS pequeño, un servidor doméstico o un equipo de la categoría Raspberry Pi.

    Solo se necesita un Mac **para las herramientas exclusivas de macOS**. Para iMessage, use [iMessage](/es/channels/imessage)
    con `imsg` en cualquier Mac que tenga una sesión iniciada en Messages; si el Gateway se ejecuta en Linux o en otro lugar,
    configure `channels.imessage.cliPath` como un contenedor SSH que ejecute `imsg` en ese Mac. Para otras
    herramientas exclusivas de macOS, ejecute el Gateway en un Mac o empareje un Node de macOS.

    Documentación: [iMessage](/es/channels/imessage), [Nodes](/es/nodes), [Modo remoto de Mac](/es/platforms/mac/remote).

  </Accordion>

  <Accordion title="¿Se necesita un Mac mini para la compatibilidad con iMessage?">
    Se necesita **algún dispositivo macOS** con una sesión iniciada en Messages; no tiene que ser un Mac mini,
    sirve cualquier Mac. Use [iMessage](/es/channels/imessage) con `imsg`; el Gateway puede ejecutarse en ese
    Mac o en otro lugar mediante un contenedor SSH `cliPath`.

    Configuraciones habituales:

    - Gateway en Linux/VPS, con `channels.imessage.cliPath` configurado como un contenedor SSH que ejecuta `imsg` en un Mac con una sesión iniciada en Messages.
    - Todo en un solo Mac para obtener la configuración más sencilla en un único equipo.

    Documentación: [iMessage](/es/channels/imessage), [Nodes](/es/nodes), [Modo remoto de Mac](/es/platforms/mac/remote).

  </Accordion>

  <Accordion title="Si compro un Mac mini para ejecutar OpenClaw, ¿puedo conectarlo a mi MacBook Pro?">
    Sí. El **Mac mini puede ejecutar el Gateway** y el MacBook Pro se conecta como un **Node**
    (dispositivo complementario). Los Nodes no ejecutan el Gateway; añaden capacidades como
    pantalla, cámara, lienzo y `system.run` en ese dispositivo.

    Patrón habitual: Gateway en el Mac mini siempre activo; el MacBook Pro ejecuta la aplicación de macOS o un
    host de Node y se empareja con el Gateway. Compruébelo con `openclaw nodes status` / `openclaw nodes list`.

    Documentación: [Nodes](/es/nodes), [CLI de Nodes](/es/cli/nodes).

  </Accordion>

  <Accordion title="¿Se puede usar Bun?">
    Se puede usar Bun para instalar dependencias o ejecutar scripts de paquetes. La CLI y el
    Gateway de OpenClaw requieren **Node** porque el almacén de estado canónico usa `node:sqlite`; Bun no
    proporciona esa API.
  </Accordion>

  <Accordion title="Telegram: ¿qué se especifica en allowFrom?">
    `channels.telegram.allowFrom` es el **identificador de usuario de Telegram del remitente humano** (numérico),
    no el nombre de usuario del bot. La configuración solo solicita identificadores numéricos de usuario; `openclaw doctor --fix`
    puede intentar resolver entradas heredadas `@username`.

    Opción más segura (sin bots de terceros): envíe un mensaje directo a su bot, ejecute `openclaw logs --follow` y lea `from.id`.

    API oficial de bots: envíe un mensaje directo a su bot, llame a `https://api.telegram.org/bot<bot_token>/getUpdates` y lea `message.from.id`.

    Terceros (menos privado): envíe un mensaje directo a `@userinfobot` o `@getidsbot`.

    Consulte [Control de acceso de Telegram](/es/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="¿Pueden varias personas usar un número de WhatsApp con distintas instancias de OpenClaw?">
    Sí, mediante el **enrutamiento multiagente**. Vincule el mensaje directo de WhatsApp de cada remitente (`peer: { kind: "direct", id: "+15551234567" }`) a un `agentId` diferente, de modo que cada persona tenga su propio espacio de trabajo y almacén de sesiones. Las respuestas seguirán procediendo de la **misma cuenta de WhatsApp**; el control de acceso a mensajes directos (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) es global por cuenta. Consulte [Enrutamiento multiagente](/es/concepts/multi-agent) y [WhatsApp](/es/channels/whatsapp).
  </Accordion>

  <Accordion title='¿Se puede ejecutar un agente de "chat rápido" y otro agente de "Opus para programación"?'>
    Sí. Use el enrutamiento multiagente: asigne a cada agente su propio modelo predeterminado y después vincule las
    rutas entrantes (cuenta del proveedor o interlocutores específicos) a cada agente. Configuración de ejemplo:
    [Enrutamiento multiagente](/es/concepts/multi-agent). Consulte también [Modelos](/es/concepts/models) y
    [Configuración](/es/gateway/configuration).
  </Accordion>

  <Accordion title="¿Funciona Homebrew en Linux?">
    Sí, mediante Linuxbrew:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Al ejecutar OpenClaw mediante systemd, asegúrese de que la variable PATH del servicio incluya
    `/home/linuxbrew/.linuxbrew/bin` (o el prefijo de brew correspondiente) para que las herramientas instaladas mediante `brew`
    se puedan resolver en shells que no sean de inicio de sesión. Las compilaciones recientes también anteponen directorios binarios habituales del usuario en los
    servicios systemd de Linux (por ejemplo, `~/.local/bin`, `~/.npm-global/bin`,
    `~/.local/share/pnpm`, `~/.bun/bin`) y respetan `PNPM_HOME`, `NPM_CONFIG_PREFIX`,
    `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` y `FNM_DIR` cuando están configurados.

  </Accordion>

  <Accordion title="Diferencia entre la instalación modificable desde Git y la instalación mediante npm">
    - **Instalación modificable (Git):** repositorio completo del código fuente, editable e ideal para colaboradores. Se compila localmente y permite modificar el código y la documentación.
    - **Instalación mediante npm:** instalación global de la CLI, sin repositorio e ideal para «simplemente ejecutarlo». Las actualizaciones se distribuyen mediante las etiquetas de distribución de npm.

    Documentación: [Primeros pasos](/es/start/getting-started), [Actualización](/es/install/updating).

  </Accordion>

  <Accordion title="¿Puedo cambiar más adelante entre instalaciones mediante npm y git?">
    Sí, con `openclaw update --channel ...` en una instalación existente. Esto **no
    elimina sus datos**; solo cambia la instalación del código de OpenClaw. El estado (`~/.openclaw`) y
    el espacio de trabajo (`~/.openclaw/workspace`) permanecen intactos.

    De npm a git:

    ```bash
    openclaw update --channel dev
    ```

    De git a npm:

    ```bash
    openclaw update --channel stable
    ```

    Añada `--dry-run` para obtener primero una vista previa del cambio de modo planificado. El actualizador ejecuta las
    acciones de seguimiento de Doctor, actualiza los orígenes de los plugins para el canal de destino y reinicia el Gateway,
    a menos que se pase `--no-restart`.

    El instalador también puede forzar cualquiera de los dos modos:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method npm
    ```

    Consejos sobre copias de seguridad: [Dónde se almacenan los elementos en el disco](/es/help/faq#where-things-live-on-disk).

  </Accordion>

  <Accordion title="¿Debo ejecutar el Gateway en mi portátil o en un VPS?">
    ¿Desea una fiabilidad 24/7? Use un **VPS**. ¿Desea reducir al mínimo las complicaciones y no le
    importan las suspensiones y los reinicios? Ejecútelo localmente.

    **Portátil (Gateway local)**

    - **Ventajas:** sin costes de servidor, acceso directo a los archivos locales y una ventana del navegador visible.
    - **Desventajas:** la suspensión o las interrupciones de red lo desconectan, las actualizaciones y los reinicios del sistema operativo lo interrumpen y el equipo debe permanecer activo.

    **VPS/nube**

    - **Ventajas:** siempre activo, red estable, sin problemas por la suspensión del portátil y más fácil de mantener en ejecución.
    - **Desventajas:** suele funcionar sin interfaz gráfica (use capturas de pantalla), solo permite acceso remoto a los archivos y requiere SSH para las actualizaciones.

    WhatsApp, Telegram, Slack, Mattermost y Discord funcionan correctamente desde un VPS; la verdadera
    disyuntiva es usar un navegador sin interfaz gráfica o una ventana visible. Consulte [Navegador](/es/tools/browser).

    Recomendación predeterminada: use un VPS si el Gateway se ha desconectado anteriormente; la ejecución local es excelente
    cuando se usa activamente el Mac y se desea acceder a los archivos locales o automatizar una interfaz
    visible del navegador.

  </Accordion>

  <Accordion title="¿Qué importancia tiene ejecutar OpenClaw en una máquina dedicada?">
    No es obligatorio, pero se recomienda para mejorar la fiabilidad y el aislamiento.

    - **Host dedicado (VPS/Mac mini/Raspberry Pi):** siempre activo, menos interrupciones por suspensión o reinicio, permisos más claros y más fácil de mantener en ejecución.
    - **Portátil o equipo de escritorio compartido:** adecuado para pruebas y uso activo, pero se producirán pausas cuando el equipo se suspenda o se actualice.

    Para obtener lo mejor de ambos entornos, mantenga el Gateway en un host dedicado y empareje el portátil como
    **Node** para las herramientas locales de pantalla, cámara y ejecución. Consulte [Nodos](/es/nodes) y [Seguridad](/es/gateway/security).

  </Accordion>

  <Accordion title="¿Cuáles son los requisitos mínimos de un VPS y el sistema operativo recomendado?">
    - **Mínimo absoluto:** 1 vCPU, 1 GB de RAM y ~500 MB de disco.
    - **Recomendado:** 1-2 vCPU y 2 GB o más de RAM para disponer de margen (registros, contenido multimedia y varios canales). Las herramientas de Node y la automatización del navegador pueden consumir muchos recursos.

    Sistema operativo: **Ubuntu LTS** (o cualquier versión moderna de Debian/Ubuntu), la ruta de instalación de Linux que más se ha probado.

    Documentación: [Linux](/es/platforms/linux), [Alojamiento en VPS](/es/vps).

  </Accordion>

  <Accordion title="¿Puedo ejecutar OpenClaw en una máquina virtual y cuáles son los requisitos?">
    Sí. Trate una máquina virtual como un VPS: debe estar siempre activa, ser accesible y tener suficiente RAM
    para el Gateway y los canales que se habiliten.

    - **Mínimo absoluto:** 1 vCPU y 1 GB de RAM.
    - **Recomendado:** 2 GB o más de RAM para varios canales, automatización del navegador o herramientas multimedia.
    - **Sistema operativo:** Ubuntu LTS u otra versión moderna de Debian/Ubuntu.

    En Windows, use **Windows Hub** para la configuración del escritorio o WSL2 para disponer de una máquina virtual de Gateway
    similar a Linux y compatible con una amplia variedad de herramientas. Consulte [Windows](/es/platforms/windows) y [Alojamiento en VPS](/es/vps).
    Para ejecutar macOS en una máquina virtual, consulte [Máquina virtual de macOS](/es/install/macos-vm).

  </Accordion>
</AccordionGroup>

## Temas relacionados

- [Preguntas frecuentes](/es/help/faq): las preguntas frecuentes principales (modelos, sesiones, Gateway, seguridad y más)
- [Descripción general de la instalación](/es/install)
- [Primeros pasos](/es/start/getting-started)
- [Solución de problemas](/es/help/troubleshooting)
