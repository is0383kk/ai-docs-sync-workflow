---
read_when:
    - Añadir una lista de verificación en BOOT.md
summary: Plantilla del espacio de trabajo para BOOT.md
title: Plantilla de BOOT.md
x-i18n:
    generated_at: "2026-07-26T04:58:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1adfb4d71f1f03716a1ddc4774a4cb6ead4b8be65bd9bb34066a9e1929a36b21
    source_path: reference/templates/BOOT.md
    workflow: 16
---

# BOOT.md

Añada aquí instrucciones de inicio breves y explícitas. El hook incluido `boot-md` ejecuta este archivo una vez por espacio de trabajo de agente cada vez que se inicia el gateway, si el archivo existe y contiene caracteres distintos de espacios en blanco. Varios agentes que comparten un espacio de trabajo solo activan una ejecución.

El hook se distribuye deshabilitado. Habilítelo primero:

```bash
openclaw hooks enable boot-md
```

Si un elemento de la lista de comprobación envía un mensaje, use la herramienta de mensajes y, a continuación, responda con el token silencioso exacto `NO_REPLY` (sin distinguir entre mayúsculas y minúsculas).

## Relacionado

- [Espacio de trabajo del agente](/es/concepts/agent-workspace)
- [Hooks](/es/automation/hooks#boot-md)
