---
read_when:
    - Se desean completados de shell para zsh/bash/fish/PowerShell
    - Es necesario almacenar en caché los scripts de completado en el estado de OpenClaw
summary: Referencia de la CLI para `openclaw completion` (generar/instalar scripts de autocompletado del shell)
title: Finalización
x-i18n:
    generated_at: "2026-07-26T05:02:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 67cb52a47036745150887c752d18e2dfa84fab2722c27c696142d23080bb2efd
    source_path: cli/completion.md
    workflow: 16
---

# `openclaw completion`

Genera scripts de completado del shell, los almacena en caché en el estado de OpenClaw y, opcionalmente, los instala en el perfil del shell.

## Uso

```bash
openclaw completion                          # imprimir el script de zsh en stdout
openclaw completion --shell fish             # imprimir el script de fish
openclaw completion --write-state            # almacenar en caché los scripts de todos los shells
openclaw completion --write-state --install  # almacenar en caché y luego instalar en un solo paso
openclaw completion --shell bash --write-state
```

## Opciones

- `-s, --shell <shell>`: shell de destino (`zsh`, `bash`, `powershell`, `fish`; valor predeterminado: `zsh`)
- `-i, --install`: instalar el completado añadiendo al perfil del shell una línea que carga el script almacenado en caché
- `--write-state`: escribir los scripts de completado en `$OPENCLAW_STATE_DIR/completions` (valor predeterminado: `~/.openclaw/completions`) sin imprimirlos en stdout; con `--shell`, escribe solo el de ese shell; de lo contrario, escribe los de los cuatro
- `-y, --yes`: omitir las solicitudes de confirmación de la instalación (modo no interactivo)

## Flujo de instalación

`--install` hace que el perfil apunte al script almacenado en caché, por lo que la caché debe existir primero: si no existe, el comando falla e indica que se debe ejecutar `openclaw completion --write-state`. Se puede combinar con `--write-state --install` para realizar ambas acciones en un solo paso. Sin `--shell`, `--install` detecta el shell a partir de `$SHELL` (y usa zsh como alternativa).

La instalación escribe un pequeño bloque `# OpenClaw Completion` en el perfil del shell y sustituye cualquier línea lenta anterior de `source <(openclaw completion ...)` por la línea que carga el script almacenado en caché:

| Shell      | Perfil                                                                                                                                                                                    |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bash       | `~/.bashrc` (usa `~/.bash_profile` como alternativa cuando falta `~/.bashrc`)                                                                                                                  |
| fish       | `~/.config/fish/config.fish`                                                                                                                                                               |
| powershell | `~/.config/powershell/Microsoft.PowerShell_profile.ps1` (en Windows: `Documents/PowerShell/Microsoft.PowerShell_profile.ps1`, o `Documents/WindowsPowerShell/...` para Windows PowerShell) |
| zsh        | `~/.zshrc`                                                                                                                                                                                 |

## Notas

- Sin `--install` ni `--write-state`, el comando imprime el script en stdout.
- La generación del completado carga de forma anticipada todo el árbol de comandos, incluidos los comandos de la CLI de los plugins, por lo que se incluyen los subcomandos anidados.
- `openclaw update` actualiza automáticamente la caché de completado después de una actualización correcta; `openclaw doctor` puede reparar configuraciones de completado ausentes u obsoletas.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
