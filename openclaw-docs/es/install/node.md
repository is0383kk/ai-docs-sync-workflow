---
read_when:
    - Debe instalar Node.js antes de instalar OpenClaw
    - Instaló OpenClaw, pero no se encuentra el comando `openclaw`
    - npm install -g falla debido a problemas de permisos o de PATH
summary: 'Instalar y configurar Node.js para OpenClaw: requisitos de versión, opciones de instalación y solución de problemas de PATH'
title: Node.js
x-i18n:
    generated_at: "2026-07-26T04:44:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef4df255c24a11a549c757b597a07b00852e60973a5e513bdcf60796037a462a
    source_path: install/node.md
    workflow: 16
---

OpenClaw requiere **Node 22.22.3+, Node 24.15+ o Node 25.9+**. **Node 24 es el entorno de ejecución predeterminado y recomendado** para instalaciones, CI y flujos de trabajo de publicación; Node 22 sigue siendo compatible mediante la línea LTS activa. Node 23 no es compatible. El [script de instalación](/es/install#alternative-install-methods) detecta e instala Node automáticamente; use esta página cuando quiera configurar Node por su cuenta (versiones, PATH, instalaciones globales).

## Comprobar la versión

```bash
node -v
```

`v24.15.0` o una versión 24.x posterior es la opción predeterminada recomendada. `v22.22.3` o una versión 22.x posterior es la ruta compatible de Node 22 LTS; Node `v25.9.0+` también es compatible. Node 23 no es compatible. Si Node no está instalado o se encuentra fuera del intervalo compatible, elija uno de los métodos de instalación siguientes.

## Instalar Node

<Tabs>
  <Tab title="macOS">
    **Homebrew** (recomendado):

    ```bash
    brew install node
    ```

    También puede descargar el instalador para macOS desde [nodejs.org](https://nodejs.org/).

  </Tab>
  <Tab title="Linux">
    **Ubuntu / Debian:**

    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt-get install -y nodejs
    ```

    **Fedora / RHEL:**

    ```bash
    sudo dnf install nodejs
    ```

    También puede usar un gestor de versiones (consulte la sección siguiente).

  </Tab>
  <Tab title="Windows">
    **winget** (recomendado):

    ```powershell
    winget install OpenJS.NodeJS.LTS
    ```

    **Chocolatey:**

    ```powershell
    choco install nodejs-lts
    ```

    También puede descargar el instalador para Windows desde [nodejs.org](https://nodejs.org/).

  </Tab>
</Tabs>

<Accordion title="Uso de un gestor de versiones (nvm, fnm, mise, asdf)">
  Los gestores de versiones permiten cambiar fácilmente entre versiones de Node. Algunas opciones populares son:

- [**fnm**](https://github.com/Schniz/fnm) - rápido y multiplataforma
- [**nvm**](https://github.com/nvm-sh/nvm) - muy utilizado en macOS/Linux
- [**mise**](https://mise.jdx.dev/) - políglota (Node, Python, Ruby, etc.)

Ejemplo con fnm:

```bash
fnm install 24
fnm use 24
```

  <Warning>
  Inicialice el gestor de versiones en el archivo de inicio del shell (`~/.zshrc` o `~/.bashrc`). Si omite este paso, es posible que `openclaw` no se encuentre en las nuevas sesiones de terminal porque PATH no incluirá el directorio bin de Node.
  </Warning>
</Accordion>

## Solución de problemas

### `openclaw: command not found`

Esto casi siempre significa que el directorio bin global de npm no está en PATH.

<Steps>
  <Step title="Buscar el prefijo global de npm">
    ```bash
    npm prefix -g
    ```
  </Step>
  <Step title="Comprobar si está en PATH">
    ```bash
    echo "$PATH"
    ```

    Busque `<npm-prefix>/bin` (macOS/Linux) o `<npm-prefix>` (Windows) en la salida.

  </Step>
  <Step title="Añadirlo al archivo de inicio del shell">
    <Tabs>
      <Tab title="macOS / Linux">
        Añádalo a `~/.zshrc` o `~/.bashrc`:

        ```bash
        export PATH="$(npm prefix -g)/bin:$PATH"
        ```

        A continuación, abra una terminal nueva (o ejecute `rehash` en zsh / `hash -r` en bash).
      </Tab>
      <Tab title="Windows">
        Añada la salida de `npm prefix -g` al PATH del sistema mediante Settings → System → Environment Variables.
      </Tab>
    </Tabs>

  </Step>
</Steps>

### Errores de permisos en `npm install -g` (Linux)

Si aparecen errores `EACCES`, cambie el prefijo global de npm a un directorio en el que el usuario tenga permisos de escritura:

```bash
mkdir -p "$HOME/.npm-global"
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
```

Añada la línea `export PATH=...` a `~/.bashrc` o `~/.zshrc` para que el cambio sea permanente.

## Contenido relacionado

- [Descripción general de la instalación](/es/install) - todos los métodos de instalación
- [Actualización](/es/install/updating) - cómo mantener OpenClaw actualizado
- [Primeros pasos](/es/start/getting-started) - pasos iniciales después de la instalación
