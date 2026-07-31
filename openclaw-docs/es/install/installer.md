---
read_when:
    - Quiere comprender `openclaw.ai/install.sh`
    - Quiere automatizar las instalaciones (CI / sin interfaz gráfica)
    - Quieres instalar desde un repositorio de GitHub clonado
summary: Cómo funcionan los scripts de instalación (install.sh, install-cli.sh, install.ps1), sus indicadores y la automatización
title: Detalles internos del instalador
x-i18n:
    generated_at: "2026-07-26T04:43:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7878f10903893b4e1902bbc79991f43edaa436bd802d5fecde41421e3e05bc2b
    source_path: install/installer.md
    workflow: 16
---

OpenClaw incluye tres scripts de instalación, disponibles desde `openclaw.ai`.

| Script                             | Plataforma             | Qué hace                                                                                   |
| ---------------------------------- | -------------------- | ---------------------------------------------------------------------------------------------- |
| [`install.sh`](#installsh)         | macOS / Linux / WSL  | Instala Node si es necesario, instala OpenClaw mediante npm (opción predeterminada) o git y puede ejecutar la configuración inicial.       |
| [`install-cli.sh`](#install-clish) | macOS / Linux / WSL  | Instala Node y OpenClaw en un prefijo local (`~/.openclaw`) mediante npm o git. No requiere acceso root. |
| [`install.ps1`](#installps1)       | Windows (PowerShell) | Instala Node si es necesario, instala OpenClaw mediante npm (opción predeterminada) o git y puede ejecutar la configuración inicial.       |

Los tres admiten Node **22.22.3+, 24.15+ o 25.9+**; Node 24 es el objetivo predeterminado para instalaciones nuevas.

## Comandos rápidos

<Tabs>
  <Tab title="install.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install-cli.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install.ps1">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```

    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -Tag beta -NoOnboard -DryRun
    ```

  </Tab>
</Tabs>

<Note>
Si la instalación se completa correctamente, pero no se encuentra `openclaw` en una terminal nueva, consulte [Solución de problemas de Node.js](/es/install/node#troubleshooting).
</Note>

---

<a id="installsh"></a>

## install.sh

<Tip>
Recomendado para la mayoría de las instalaciones interactivas en macOS/Linux/WSL.
</Tip>

### Flujo (install.sh)

<Steps>
  <Step title="Detectar el sistema operativo">
    Admite macOS y Linux (incluido WSL).
  </Step>
  <Step title="Garantizar Node.js 24 de forma predeterminada">
    Comprueba la versión de Node e instala Node 24 si es necesario (Homebrew en macOS y scripts de configuración de NodeSource en Linux con apt/dnf/yum). En macOS, Homebrew solo se instala cuando el instalador lo necesita para Node o Git. Se admiten Node 22.22.3+, Node 24.15+ y Node 25.9+; Node 23 no es compatible.
    En Alpine/Linux con musl, el instalador utiliza paquetes apk en lugar de NodeSource y verifica la versión real de SQLite enlazada. Las fuentes de paquetes estables actuales de Alpine pueden proporcionar una versión de Node suficientemente nueva con una versión vulnerable de SQLite del sistema; cuando esto ocurra, utilice en su lugar un contenedor oficial `node:24-alpine` o un host basado en glibc.
  </Step>
  <Step title="Garantizar Git">
    Instala Git si falta mediante el gestor de paquetes detectado, incluidos Homebrew en macOS y apk en Alpine.
  </Step>
  <Step title="Instalar OpenClaw">
    - Método `npm` (predeterminado): instalación global mediante npm
    - Método `git`: clona o actualiza el repositorio, instala las dependencias con pnpm, compila y, a continuación, instala el contenedor de comandos en `~/.local/bin/openclaw`

  </Step>
  <Step title="Tareas posteriores a la instalación">
    - Resuelve el binario `openclaw` recién instalado para los comandos posteriores
    - En una instalación sin configurar, inicia la configuración inicial antes de las comprobaciones de doctor o del Gateway. Con `--no-onboard` o sin TTY, muestra el comando para completar la configuración más adelante.
    - En una instalación configurada, actualiza y reinicia, en la medida de lo posible, un servicio Gateway cargado y ejecuta doctor. Las actualizaciones ponen al día los plugins cuando es posible o muestran el comando manual en una ejecución sin interfaz con las solicitudes habilitadas.
    - Cuando se ejecuta `--verify`, comprueba la versión instalada y solo comprueba el estado del Gateway después de que exista la configuración.

  </Step>
</Steps>

### Detección de una copia de trabajo del código fuente

Si se ejecuta dentro de una copia de trabajo de OpenClaw (`package.json` + `pnpm-workspace.yaml`), el script ofrece:

- utilizar la copia de trabajo (`git`), o
- utilizar la instalación global (`npm`)

Si no hay ningún TTY disponible y no se ha establecido ningún método de instalación, utiliza `npm` de forma predeterminada y muestra una advertencia.

El script finaliza con el código `2` si la selección del método no es válida o si los valores de `--install-method` no son válidos.

### Ejemplos (install.sh)

<Tabs>
  <Tab title="Predeterminado">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="Omitir la configuración inicial">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```
  </Tab>
  <Tab title="Instalación mediante Git">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```
  </Tab>
  <Tab title="Copia de trabajo de la rama principal de GitHub">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --version main
    ```
  </Tab>
  <Tab title="Simulación">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --dry-run
    ```
  </Tab>
  <Tab title="Verificar después de la instalación">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard --verify
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Referencia de opciones">

| Opción                                    | Descripción                                                             |
| --------------------------------------- | ----------------------------------------------------------------------- |
| `--install-method \| --method npm\|git` | Selecciona el método de instalación (predeterminado: `npm`)                                  |
| `--npm`                                 | Atajo para el método npm                                                 |
| `--git \| --github`                     | Atajo para el método git                                                 |
| `--version <version\|dist-tag\|spec>`   | Versión, etiqueta de distribución o especificación de paquete de npm (predeterminado: `latest`)              |
| `--beta`                                | Utiliza la etiqueta de distribución beta si está disponible; de lo contrario, recurre a `latest`              |
| `--git-dir \| --dir <path>`             | Directorio de la copia de trabajo (predeterminado: `~/openclaw`)                              |
| `--no-git-update`                       | Omite `git pull` para una copia de trabajo existente                                   |
| `--no-prompt`                           | Desactiva las solicitudes                                                         |
| `--no-onboard`                          | Omite la configuración inicial                                                         |
| `--onboard`                             | Habilita la configuración inicial                                                       |
| `--verify`                              | Ejecuta una verificación rápida posterior a la instalación (`--version` y estado del Gateway si está cargado) |
| `--dry-run`                             | Muestra las acciones sin aplicar cambios                                  |
| `--verbose`                             | Habilita la salida de depuración (`set -x` y registros de npm de nivel notice)                   |
| `--help \| -h`                          | Muestra el uso                                                              |

  </Accordion>

  <Accordion title="Referencia de variables de entorno">

| Variable                                          | Descripción                                                        |
| ------------------------------------------------- | ------------------------------------------------------------------ |
| `OPENCLAW_INSTALL_METHOD=git\|npm`                | Método de instalación                                                     |
| `OPENCLAW_VERSION=latest\|next\|<semver>\|<spec>` | Versión, etiqueta de distribución o especificación de paquete de npm                             |
| `OPENCLAW_BETA=0\|1`                              | Utiliza beta si está disponible                                              |
| `OPENCLAW_HOME=<path>`                            | Directorio base para el estado de OpenClaw y las rutas predeterminadas de git/configuración inicial |
| `OPENCLAW_GIT_DIR=<path>`                         | Directorio de la copia de trabajo                                                 |
| `OPENCLAW_GIT_UPDATE=0\|1`                        | Activa o desactiva las actualizaciones de git                                                 |
| `OPENCLAW_NO_PROMPT=1`                            | Desactiva las solicitudes                                                    |
| `OPENCLAW_VERIFY_INSTALL=1`                       | Ejecuta la verificación rápida posterior a la instalación                                  |
| `OPENCLAW_NO_ONBOARD=1`                           | Omite la configuración inicial                                                    |
| `OPENCLAW_DRY_RUN=1`                              | Modo de simulación                                                       |
| `OPENCLAW_VERBOSE=1`                              | Modo de depuración                                                         |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice`       | Nivel de registro de npm (predeterminado: `error`; oculta el ruido de obsolescencia de npm)      |

  </Accordion>
</AccordionGroup>

---

<a id="install-clish"></a>

## install-cli.sh

<Info>
Diseñado para entornos en los que se desea almacenar todo bajo un prefijo local
(valor predeterminado: `~/.openclaw`) y no depender de una instalación de Node en el sistema. Admite instalaciones mediante npm
de forma predeterminada, además de instalaciones desde una copia de trabajo de git con el mismo flujo de prefijo.
</Info>

### Flujo (install-cli.sh)

<Steps>
  <Step title="Instalar el entorno de ejecución local de Node">
    Descarga un archivo tar de una versión LTS compatible y fijada de Node (la versión está integrada en el script y se actualiza de forma independiente; valor predeterminado: `24.15.0`) en `<prefix>/tools/node-v<version>` y verifica su SHA-256.
    Linux ARMv7 utiliza Node `22.22.3` porque no hay binarios oficiales de Node 24+ disponibles para ARMv7.
    En Alpine/Linux con musl, donde Node no publica archivos tar compatibles con el entorno de ejecución fijado, instala `nodejs` y `npm` con `apk` y, a continuación, verifica tanto Node como la biblioteca SQLite realmente enlazada. Las fuentes de paquetes estables actuales de Alpine todavía pueden enlazar una versión vulnerable de SQLite incluso con una versión de Node suficientemente nueva; utilice un contenedor oficial `node:24-alpine` o un host basado en glibc cuando la comprobación de seguridad rechace el paquete.
  </Step>
  <Step title="Garantizar Git">
    Si falta Git, intenta instalarlo mediante apt/dnf/yum/apk en Linux o Homebrew en macOS.
  </Step>
  <Step title="Instalar OpenClaw bajo el prefijo">
    - Método `npm` (predeterminado): instala bajo el prefijo mediante npm y, a continuación, escribe el contenedor de comandos en `<prefix>/bin/openclaw`
    - Método `git`: clona o actualiza una copia de trabajo (valor predeterminado: `~/openclaw`) y también escribe el contenedor de comandos en `<prefix>/bin/openclaw`

  </Step>
  <Step title="Actualizar el servicio Gateway cargado">
    Si ya hay un servicio Gateway cargado desde ese mismo prefijo, el script ejecuta
    `openclaw gateway install --force`, que activa el servicio de sustitución,
    y, a continuación, comprueba el estado del Gateway en la medida de lo posible.
  </Step>
</Steps>

### Ejemplos (install-cli.sh)

<Tabs>
  <Tab title="Predeterminado">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```
  </Tab>
  <Tab title="Prefijo y versión personalizados">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --prefix /opt/openclaw --version latest
    ```
  </Tab>
  <Tab title="Instalación mediante Git">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --install-method git --git-dir ~/openclaw
    ```
  </Tab>
  <Tab title="Salida JSON para automatización">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="Ejecutar la configuración inicial">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --onboard
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Referencia de opciones">

| Indicador                               | Descripción                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------- |
| `--prefix <path>`                       | Prefijo de instalación (valor predeterminado: `~/.openclaw`)                                         |
| `--install-method \| --method npm\|git` | Elegir el método de instalación (valor predeterminado: `npm`)                                          |
| `--npm`                                 | Atajo para el método npm                                                         |
| `--git \| --github`                     | Atajo para el método git                                                         |
| `--git-dir \| --dir <path>`             | Directorio de checkout de Git (valor predeterminado: `~/openclaw`)                                  |
| `--version <ver>`                       | Versión o dist-tag de OpenClaw (valor predeterminado: `latest`)                                |
| `--node-version <ver>`                  | Versión de Node (valor predeterminado: `24.15.0`; `22.22.3` en Linux ARMv7)                     |
| `--json`                                | Emitir eventos NDJSON                                                              |
| `--onboard`                             | Ejecutar `openclaw onboard` después de la instalación                                            |
| `--no-onboard`                          | Omitir la incorporación (valor predeterminado)                                                       |
| `--set-npm-prefix`                      | En Linux, forzar el prefijo de npm a `~/.npm-global` si no se puede escribir en el prefijo actual |
| `--help \| -h`                          | Mostrar el uso                                                                      |

  </Accordion>

  <Accordion title="Referencia de variables de entorno">

| Variable                                    | Descripción                                                        |
| ------------------------------------------- | ------------------------------------------------------------------ |
| `OPENCLAW_PREFIX=<path>`                    | Prefijo de instalación                                                     |
| `OPENCLAW_INSTALL_METHOD=git\|npm`          | Método de instalación                                                     |
| `OPENCLAW_VERSION=<ver>`                    | Versión o dist-tag de OpenClaw                                       |
| `OPENCLAW_NODE_VERSION=<ver>`               | Versión de Node                                                       |
| `OPENCLAW_HOME=<path>`                      | Directorio base para el estado de OpenClaw y las rutas predeterminadas de git/incorporación |
| `OPENCLAW_GIT_DIR=<path>`                   | Directorio de checkout de Git para instalaciones mediante git                            |
| `OPENCLAW_GIT_UPDATE=0\|1`                  | Activar o desactivar las actualizaciones de git para checkouts existentes                          |
| `OPENCLAW_NO_ONBOARD=1`                     | Omitir la incorporación                                                    |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice` | Nivel de registro de npm (valor predeterminado: `error`)                                   |

  </Accordion>
</AccordionGroup>

<Note>
`openclaw@main` y otras especificaciones de origen de GitHub no son destinos `--version` válidos para instalaciones mediante npm. Use `--install-method git --version main` en su lugar.
</Note>

---

<a id="installps1"></a>

## install.ps1

### Flujo (install.ps1)

<Steps>
  <Step title="Garantizar un entorno de PowerShell y Windows">
    Requiere PowerShell 5+.
  </Step>
  <Step title="Garantizar Node.js 24 de forma predeterminada">
    Si no está disponible, intenta instalarlo mediante winget, después Chocolatey y, por último, Scoop. Si no hay ningún gestor de paquetes disponible, el script descarga el zip oficial de Node.js 24 para Windows en `%LOCALAPPDATA%\OpenClaw\deps\portable-node` y lo añade al PATH del proceso actual y del usuario. Se admiten Node 22.22.3+, Node 24.15+ y Node 25.9+; Node 23 no es compatible.
  </Step>
  <Step title="Instalar OpenClaw">
    - Método `npm` (predeterminado): instalación global de npm mediante el `-Tag` seleccionado, iniciada desde un directorio temporal del instalador con permisos de escritura para que las consolas abiertas en carpetas protegidas como `C:\` sigan funcionando
    - Método `git`: clona/actualiza el repositorio, instala/compila con pnpm e instala el contenedor en `%USERPROFILE%\.local\bin\openclaw.cmd`. Si Git no está disponible, el script instala MinGit localmente para el usuario en `%LOCALAPPDATA%\OpenClaw\deps\portable-git` y lo añade al PATH del proceso actual y del usuario.

  </Step>
  <Step title="Tareas posteriores a la instalación">
    - Añade el directorio bin necesario al PATH del usuario cuando es posible
    - Actualiza, en la medida de lo posible, un servicio Gateway cargado (`openclaw gateway install --force` y, después, reinicio)
    - Ejecuta `openclaw doctor --non-interactive` en actualizaciones e instalaciones mediante git (en la medida de lo posible)

  </Step>
  <Step title="Gestionar errores">
    Las instalaciones mediante `iwr ... | iex` y bloques de script notifican un error de terminación sin cerrar la sesión actual de PowerShell. Las instalaciones directas mediante `powershell -File` / `pwsh -File` siguen finalizando con un código distinto de cero para permitir la automatización.
  </Step>
</Steps>

### Ejemplos (install.ps1)

<Tabs>
  <Tab title="Predeterminado">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```
  </Tab>
  <Tab title="Instalación mediante Git">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git
    ```
  </Tab>
  <Tab title="Checkout de la rama main de GitHub">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git -Tag main
    ```
  </Tab>
  <Tab title="Directorio de git personalizado">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git -GitDir "C:\openclaw"
    ```
  </Tab>
  <Tab title="Ejecución de prueba">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -DryRun
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Referencia de indicadores">

| Indicador                   | Descripción                                                |
| --------------------------- | ---------------------------------------------------------- |
| `-InstallMethod npm\|git`   | Método de instalación (valor predeterminado: `npm`)                            |
| `-Tag <tag\|version\|spec>` | dist-tag, versión o especificación de paquete de npm (valor predeterminado: `latest`) |
| `-GitDir <path>`            | Directorio de checkout (valor predeterminado: `%USERPROFILE%\openclaw`)     |
| `-NoOnboard`                | Omitir la incorporación                                            |
| `-NoGitUpdate`              | Omitir `git pull`                                            |
| `-DryRun`                   | Mostrar solo las acciones                                         |

  </Accordion>

  <Accordion title="Referencia de variables de entorno">

| Variable                           | Descripción            |
| ---------------------------------- | ---------------------- |
| `OPENCLAW_INSTALL_METHOD=git\|npm` | Método de instalación  |
| `OPENCLAW_GIT_DIR=<path>`          | Directorio de checkout |
| `OPENCLAW_NO_ONBOARD=1`            | Omitir la incorporación |
| `OPENCLAW_GIT_UPDATE=0`            | Desactivar git pull    |
| `OPENCLAW_DRY_RUN=1`               | Modo de ejecución de prueba |

  </Accordion>
</AccordionGroup>

<Note>
Si se usa `-InstallMethod git` y Git no está disponible, el script intenta instalar MinGit localmente para el usuario antes de mostrar el enlace de Git for Windows.
</Note>

---

## CI y automatización

Use indicadores o variables de entorno no interactivos para obtener ejecuciones predecibles.

<Tabs>
  <Tab title="install.sh (npm no interactivo)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-prompt --no-onboard
    ```
  </Tab>
  <Tab title="install.sh (git no interactivo)">
    ```bash
    OPENCLAW_INSTALL_METHOD=git OPENCLAW_NO_PROMPT=1 \
      curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="install-cli.sh (JSON)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="install.ps1 (omitir incorporación)">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```
  </Tab>
</Tabs>

---

## Solución de problemas

<AccordionGroup>
  <Accordion title="¿Por qué se necesita Git?">
    Git es necesario para el método de instalación `git`. Para las instalaciones `npm`, Git también se comprueba/instala para evitar errores `spawn git ENOENT` cuando las dependencias usan URL de git.
  </Accordion>

  <Accordion title="¿Por qué npm genera EACCES en Linux?">
    Algunas configuraciones de Linux dirigen el prefijo global de npm a rutas propiedad de root. `install.sh` puede cambiar el prefijo a `~/.npm-global` y añadir exportaciones de PATH a los archivos rc de la consola (cuando dichos archivos existen).
  </Accordion>

  <Accordion title='Windows: "npm error spawn git / ENOENT"'>
    Vuelva a ejecutar el instalador para que pueda instalar MinGit localmente para el usuario, o instale Git for Windows y vuelva a abrir PowerShell.
  </Accordion>

  <Accordion title='Windows: "openclaw is not recognized"'>
    Ejecute `npm config get prefix` y añada ese directorio al PATH del usuario (no se necesita el sufijo `\bin` en Windows); después, vuelva a abrir PowerShell.
  </Accordion>

  <Accordion title="Windows: cómo obtener resultados detallados del instalador">
    `install.ps1` no ofrece un modificador `-Verbose`.
    Use el seguimiento de PowerShell para obtener diagnósticos a nivel de script:

    ```powershell
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

  </Accordion>

  <Accordion title="openclaw no se encuentra después de la instalación">
    Suele deberse a un problema con PATH. Consulte [Solución de problemas de Node.js](/es/install/node#troubleshooting).
  </Accordion>
</AccordionGroup>

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Actualización](/es/install/updating)
- [Desinstalación](/es/install/uninstall)
