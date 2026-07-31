---
read_when:
    - Inicialización manual de un espacio de trabajo
summary: Plantilla del espacio de trabajo para HEARTBEAT.md
title: Plantilla de HEARTBEAT.md
x-i18n:
    generated_at: "2026-07-26T05:28:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# Plantilla de HEARTBEAT.md

`HEARTBEAT.md` se encuentra en el espacio de trabajo del agente y contiene la lista de comprobación periódica de Heartbeat. Manténgalo vacío, o únicamente con espacios en blanco, comentarios de Markdown, encabezados ATX, estructuras de lista vacías (`- `, `* [ ]`) o marcadores de bloque, para que OpenClaw omita por completo la llamada al modelo de Heartbeat (`reason=empty-heartbeat-file`).

Contenido predeterminado incluido:

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# Mantenga este archivo vacío (o únicamente con comentarios) para omitir las llamadas de Heartbeat a la API.

# Añada a continuación una breve lista de comprobación cuando Heartbeat deba inspeccionar el contexto compartido.
```

Añada una breve lista de comprobación debajo de las líneas de comentarios solo cuando un turno de Heartbeat deba inspeccionar los elementos conjuntamente. Manténgala breve: las ejecuciones de Heartbeat leen este archivo en cada intervalo (de forma predeterminada, cada 30 minutos), por lo que las instrucciones excesivas consumen tokens cada vez que se activa.

Para comprobaciones programadas de forma independiente o que solo se ejecuten cuando corresponda, cree [tareas de Cron](/es/automation/cron-jobs). El borrador de Heartbeat ya no admite la sintaxis del programador. Ejecute `openclaw doctor --fix` para convertir los bloques `tasks:` antiguos.

## Contenido relacionado

- [Heartbeat](/es/gateway/heartbeat)
- [Configuración de Heartbeat](/es/gateway/config-agents)
