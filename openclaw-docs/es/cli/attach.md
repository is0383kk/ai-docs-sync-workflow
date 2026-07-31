---
read_when:
    - Quiere que Claude Code use las herramientas MCP de OpenClaw Gateway
    - Necesita una autorización MCP temporal vinculada a la sesión para un arnés externo
summary: Referencia de la CLI para `openclaw attach` (iniciar Claude Code con una concesión de MCP del Gateway de alcance limitado)
title: Adjuntar CLI
x-i18n:
    generated_at: "2026-07-26T04:33:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d8ac60724adef1439af09179806af537b8f2925f06b3715850e4dd3b83b080f
    source_path: cli/attach.md
    workflow: 16
---

`openclaw attach` inicia Claude Code con una configuración temporal estricta de MCP vinculada a una sesión de Gateway.

```sh
openclaw attach
openclaw attach --session agent:main:telegram:123 --ttl 600000
openclaw attach --print-config
```

Opciones:

- `--session <key>` vincula la concesión a una sesión de Gateway. El valor predeterminado es la sesión principal.
- `--ttl <ms>` solicita un TTL positivo para la concesión, en milisegundos. El Gateway aplica su propio límite máximo.
- `--bin <path>` selecciona el binario de Claude Code. Valor predeterminado: `claude`.
- `--print-config` escribe el archivo temporal `.mcp.json`, muestra el comando de inicio y las variables de entorno, y mantiene activa la concesión hasta que vence el TTL (no inicia Claude Code ni revoca la concesión).

El token de portador se pasa mediante variables de entorno, no mediante argv. OpenClaw inicia Claude Code con `--strict-mcp-config --mcp-config <path>` para que los servidores MCP ambientales de Claude no se unan a la sesión vinculada. Los inicios normales (sin `--print-config`) revocan la concesión cuando finaliza el proceso de Claude Code.

Véase también: [CLI de Gateway](/es/cli/gateway), [CLI de MCP](/es/cli/mcp) y [CLI de ACP](/es/cli/acp).
