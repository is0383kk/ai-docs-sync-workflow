---
read_when:
    - Se buscan instalaciones reproducibles y reversibles
    - Ya se utiliza Nix/NixOS/Home Manager
    - Se desea que todo esté fijado y gestionado de forma declarativa
summary: Instalar OpenClaw de forma declarativa con Nix
title: Nix
x-i18n:
    generated_at: "2026-07-26T05:44:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f74e259ec3d909c73d9184db24d236135db04c29c2e7fab9be9e6fa7f98ba91
    source_path: install/nix.md
    workflow: 16
---

Instala OpenClaw de forma declarativa con **[nix-openclaw](https://github.com/openclaw/nix-openclaw)**, el módulo oficial y completo de Home Manager.

<Info>
El repositorio [nix-openclaw](https://github.com/openclaw/nix-openclaw) es la fuente de referencia para la instalación con Nix. Esta página ofrece una descripción general rápida.
</Info>

## Qué se obtiene

- Gateway + aplicación para macOS + herramientas (whisper, spotify, cámaras), todo con versiones fijadas
- Servicio launchd que se mantiene tras los reinicios
- Sistema de Plugins con configuración declarativa
- Reversión instantánea: `home-manager switch --rollback`

## Inicio rápido

<Steps>
  <Step title="Instalar Determinate Nix">
    Si Nix aún no está instalado, sigue las instrucciones del [instalador de Determinate Nix](https://github.com/DeterminateSystems/nix-installer).
  </Step>
  <Step title="Crear un flake local">
    Utiliza la plantilla orientada a agentes del repositorio nix-openclaw:
    ```bash
    mkdir -p ~/code/openclaw-local
    # Copia templates/agent-first/flake.nix del repositorio nix-openclaw
    ```
  </Step>
  <Step title="Configurar los secretos">
    Configura el token del bot de mensajería y la clave de API del proveedor del modelo. Los archivos de texto sin formato en `~/.secrets/` funcionan correctamente.
  </Step>
  <Step title="Completar los marcadores de posición de la plantilla y aplicar">
    ```bash
    home-manager switch
    ```
  </Step>
  <Step title="Verificar">
    Confirma que el servicio launchd esté en ejecución y que el bot responda a los mensajes.
  </Step>
</Steps>

Consulta el [README de nix-openclaw](https://github.com/openclaw/nix-openclaw) para ver todas las opciones del módulo y ejemplos.

## Comportamiento del entorno de ejecución en modo Nix

Cuando se establece `OPENCLAW_NIX_MODE=1` (automáticamente con nix-openclaw), OpenClaw entra en un modo determinista para las instalaciones gestionadas con Nix. Otros paquetes de Nix pueden establecer el mismo modo; nix-openclaw es la referencia oficial.

También se puede establecer manualmente:

```bash
export OPENCLAW_NIX_MODE=1
```

En macOS, la aplicación con interfaz gráfica no hereda las variables de entorno del shell. En su lugar, habilita el modo Nix mediante `defaults`:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### Qué cambia en el modo Nix

- Los flujos de instalación automática y automodificación están deshabilitados.
- `openclaw.json` se considera inmutable. Los valores predeterminados derivados del inicio permanecen únicamente en el entorno de ejecución, y las operaciones que escriben la configuración (configuración inicial, incorporación, modificación de `openclaw update`, instalación, actualización, desinstalación y habilitación de Plugins, `doctor --fix`, `doctor --generate-gateway-token`, `openclaw config set`) se niegan a editar el archivo.
- Edita en su lugar el código fuente de Nix. Para nix-openclaw, utiliza el [inicio rápido](https://github.com/openclaw/nix-openclaw#quick-start) orientado a agentes y establece la configuración en `programs.openclaw.config` o `instances.<name>.config`.
- Las dependencias que faltan muestran mensajes de solución específicos de Nix.
- La interfaz de usuario muestra un aviso de modo Nix de solo lectura.

### Rutas de configuración y estado

OpenClaw lee la configuración JSON5 de `OPENCLAW_CONFIG_PATH` y almacena los datos modificables en `OPENCLAW_STATE_DIR`. Con Nix, establece estas rutas explícitamente en ubicaciones gestionadas por Nix para que el estado del entorno de ejecución y la configuración permanezcan fuera del almacén inmutable.

| Variable               | Valor predeterminado                    |
| ---------------------- | --------------------------------------- |
| `OPENCLAW_HOME`        | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR`   | `~/.openclaw`                           |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json`     |

### Detección de PATH para el servicio

El servicio Gateway de launchd/systemd detecta automáticamente los ejecutables de los perfiles de Nix para que los Plugins y las herramientas que invocan ejecutables instalados mediante `nix` funcionen sin configurar PATH manualmente:

- Cuando se establece `NIX_PROFILES`, cada entrada se añade al PATH del servicio con precedencia de derecha a izquierda (coincide con la precedencia del shell de Nix: la entrada situada más a la derecha prevalece).
- Cuando no se establece `NIX_PROFILES`, se añade `~/.nix-profile/bin` como alternativa.

Esto se aplica tanto a los entornos de servicio launchd de macOS como a los de systemd de Linux.

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="nix-openclaw" href="https://github.com/openclaw/nix-openclaw" icon="arrow-up-right-from-square">
    Módulo de Home Manager que constituye la fuente de referencia y guía completa de configuración.
  </Card>
  <Card title="Asistente de configuración" href="/es/start/wizard" icon="wand-magic-sparkles">
    Guía paso a paso para la configuración mediante la CLI sin Nix.
  </Card>
  <Card title="Docker" href="/es/install/docker" icon="docker">
    Configuración en contenedores como alternativa sin Nix.
  </Card>
  <Card title="Actualización" href="/es/install/updating" icon="arrow-up-right-from-square">
    Actualización de las instalaciones gestionadas por Home Manager junto con el paquete.
  </Card>
</CardGroup>
