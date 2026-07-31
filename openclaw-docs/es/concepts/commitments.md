---
read_when:
    - Se está actualizando una configuración que utilizaba compromisos inferidos
    - Se desea consultar o descartar registros de seguimiento almacenados anteriormente
sidebarTitle: Commitments
summary: Orientación sobre el estado y la limpieza de compromisos de seguimiento inferidos retirados
title: Compromisos inferidos
x-i18n:
    generated_at: "2026-07-26T04:35:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

El experimento de compromisos inferidos se ha retirado. OpenClaw ya no extrae nuevos
seguimientos de conversaciones ni los entrega mediante Heartbeat, y el antiguo
bloque de configuración `commitments` se elimina mediante `openclaw doctor --fix`.

Los recordatorios exactos y el trabajo programado continúan utilizando las
[tareas programadas](/es/automation/cron-jobs). Los hechos conversacionales persistentes deben almacenarse en la
[memoria](/es/concepts/memory).

## Registros existentes

Los compromisos almacenados anteriormente permanecen en la base de datos de estado SQLite compartida para que una
actualización no destruya el historial visible para el operador. Utilice la CLI de mantenimiento
heredada para inspeccionar o descartar esas filas:

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

Consulte [`openclaw commitments`](/es/cli/commitments) para obtener la referencia del comando
de mantenimiento.

## Contenido relacionado

- [Tareas programadas](/es/automation/cron-jobs)
- [Descripción general de la memoria](/es/concepts/memory)
- [Heartbeat](/es/gateway/heartbeat)
