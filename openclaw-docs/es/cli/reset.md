---
read_when:
    - Se desea borrar el estado local y mantener instalada la CLI
    - Se desea una simulación de lo que se eliminaría
summary: Referencia de la CLI para `openclaw reset` (restablecer el estado/la configuración local)
title: Restablecer
x-i18n:
    generated_at: "2026-07-26T05:07:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54f1d320ee368dae4a4bfb32dea73d19eb35f9f30edd12d9c2580ab7e6a26fa6
    source_path: cli/reset.md
    workflow: 16
---

# `openclaw reset`

Restablece la configuración y el estado locales (mantiene instalada la CLI).

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

## Opciones

- `--scope <scope>`: `config`, `config+creds+sessions` o `full`
- `--yes`: omite las solicitudes de confirmación
- `--non-interactive`: deshabilita las solicitudes; requiere `--scope` y `--yes`
- `--dry-run`: muestra las acciones sin eliminar archivos

## Ámbitos

| Ámbito                  | Elimina                                                                               | Detiene primero el Gateway |
| ----------------------- | ------------------------------------------------------------------------------------- | -------------------------- |
| `config`      | solo el archivo de configuración                                                      | no                         |
| `config+creds+sessions`      | el archivo de configuración, el directorio de OAuth/credenciales y los directorios de sesiones de cada agente | sí |
| `full`      | el directorio de estado (incluida la base de datos SQLite compartida) y los directorios de espacios de trabajo | sí |

`config+creds+sessions` y `full` detienen un servicio administrado del Gateway que esté en ejecución antes de eliminar el estado.

## Notas

- Ejecuta primero `openclaw backup create` para obtener una instantánea restaurable antes de eliminar el estado local.
- El estado de configuración del espacio de trabajo y las atestaciones son filas de la base de datos SQLite compartida, por lo que `full` los elimina junto con el directorio de estado; actualmente no hay archivos auxiliares de atestación que deban eliminarse por separado.
- Sin `--scope`, `openclaw reset` solicita de forma interactiva el ámbito que se eliminará.
- `--non-interactive` solo es válido cuando están definidos tanto `--scope` como `--yes`.
- `config+creds+sessions` y `full` muestran `Next: openclaw onboard --install-daemon` al finalizar.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
