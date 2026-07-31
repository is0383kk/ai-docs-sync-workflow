---
read_when:
    - Encuentra `openclaw flows` en documentación antigua o notas de la versión
    - Se necesita una referencia rápida para inspeccionar TaskFlow
summary: 'Redirección: los comandos de flujo se encuentran en `openclaw tasks flow`'
title: Flujos (redirección)
x-i18n:
    generated_at: "2026-07-26T04:33:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 05d27154190d6087649612d81ce15f0cbc9459aa89ab22211582c18f4fc2943c
    source_path: cli/flows.md
    workflow: 16
---

# `openclaw tasks flow`

No existe ningún comando de nivel superior `openclaw flows`. La inspección persistente de TaskFlow se encuentra en `openclaw tasks flow`.

## Subcomandos

```bash
openclaw tasks flow list   [--json] [--status <name>]
openclaw tasks flow show   <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

| Subcomando | Descripción                | Argumentos / opciones                                                                   |
| ---------- | -------------------------- | ------------------------------------------------------------------------------------- |
| `list`     | Enumera los TaskFlow rastreados.    | Salida legible por máquina `--json`; filtro `--status <name>` (consulte los valores de estado a continuación). |
| `show`     | Muestra un TaskFlow.         | Identificador de flujo o clave de propietario `<lookup>`; salida legible por máquina `--json`.                    |
| `cancel`   | Cancela un TaskFlow en ejecución. | Identificador de flujo o clave de propietario `<lookup>`.                                                      |

`<lookup>` acepta un identificador de flujo (devuelto por `list` / `show`) o la clave de propietario del flujo (el identificador estable que utiliza el subsistema propietario para realizar el seguimiento del flujo).

### Valores del filtro de estado

`--status` en `list` acepta uno de los siguientes valores: `queued`, `running`, `waiting`, `blocked`, `succeeded`, `failed`, `cancelled`, `lost`.

## Ejemplos

```bash
openclaw tasks flow list
openclaw tasks flow list --status running
openclaw tasks flow list --json
openclaw tasks flow show flow_abc123
openclaw tasks flow show flow_abc123 --json
openclaw tasks flow cancel flow_abc123
```

Para consultar los conceptos y la creación de TaskFlow, consulte [TaskFlow](/es/automation/taskflow). Para el comando principal `tasks`, consulte la [referencia de la CLI de tareas](/es/cli/tasks).

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Automatización](/es/automation)
- [TaskFlow](/es/automation/taskflow)
