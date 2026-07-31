---
read_when:
    - Inicialización manual de un espacio de trabajo
summary: Plantilla del espacio de trabajo para TOOLS.md
title: Plantilla de TOOLS.md
x-i18n:
    generated_at: "2026-07-26T05:21:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20eab78b3b117566a1d33a70873e70ff2d5099543aa44e2719dc8d0797099afe
    source_path: reference/templates/TOOLS.md
    workflow: 16
---

# TOOLS.md - Notas locales

Las Skills definen _cómo_ funcionan las herramientas. Este archivo contiene _tus_ datos específicos: aquello que es exclusivo de tu configuración, como los nombres y las ubicaciones de las cámaras, los hosts y alias SSH, las voces TTS preferidas, los nombres de altavoces y habitaciones, los apodos de los dispositivos y cualquier dato específico del entorno.

## Ejemplos

```markdown
### Cámaras

- living-room → Zona principal, gran angular de 180°
- front-door → Entrada, activada por movimiento

### SSH

- home-server → 192.168.1.100, usuario: admin

### TTS

- Voz preferida: "Nova" (cálida, ligeramente británica)
- Altavoz predeterminado: Kitchen HomePod
```

## ¿Por qué mantenerlos separados?

Las Skills se comparten. Tu configuración es tuya. Mantenerlas separadas permite actualizar las Skills sin perder tus notas y compartirlas sin revelar tu infraestructura.

---

Añade todo lo que te ayude a realizar tu trabajo. Esta es tu hoja de referencia rápida.

## Temas relacionados

- [Espacio de trabajo del agente](/es/concepts/agent-workspace)
