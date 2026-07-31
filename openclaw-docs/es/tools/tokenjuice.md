---
read_when:
    - Quieres resultados más breves de las herramientas `exec` o `bash` en OpenClaw
    - Quiere instalar o habilitar el plugin Tokenjuice
    - Necesita comprender qué cambia Tokenjuice y qué deja sin procesar.
summary: Compacta los resultados ruidosos de las herramientas exec y bash con el Plugin opcional Tokenjuice
title: Tokenjuice
x-i18n:
    generated_at: "2026-07-26T05:02:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96b110563a2600429dd9f0d38997cf7cc5ae4952b7f146a6ab64c96f2f202440
    source_path: tools/tokenjuice.md
    workflow: 16
---

`tokenjuice` es un plugin externo opcional que compacta los resultados ruidosos de las herramientas `exec` y `bash`
después de que el comando ya se haya ejecutado.

Cambia el `tool_result` devuelto, no el comando en sí. Tokenjuice no
reescribe la entrada del shell, vuelve a ejecutar comandos ni cambia los códigos de salida.

Actualmente, esto se aplica a las ejecuciones integradas de OpenClaw y a las herramientas dinámicas de OpenClaw en el
entorno de pruebas del servidor de aplicaciones de Codex. Tokenjuice se conecta al middleware de resultados de herramientas de OpenClaw y
recorta la salida antes de que vuelva a la sesión activa del entorno de pruebas.

## Activar el plugin

Instalar una vez:

```bash
openclaw plugins install clawhub:@openclaw/tokenjuice
```

A continuación, activarlo:

```bash
openclaw config set plugins.entries.tokenjuice.enabled true
```

Equivalente:

```bash
openclaw plugins enable tokenjuice
```

Si se prefiere editar la configuración directamente:

```json5
{
  plugins: {
    entries: {
      tokenjuice: {
        enabled: true,
      },
    },
  },
}
```

## Qué cambia tokenjuice

- Compacta los resultados ruidosos de `exec` y `bash` antes de que se vuelvan a introducir en la sesión.
- Mantiene intacta la ejecución del comando original.
- Aplica una política de inventario seguro: las lecturas exactas del contenido de archivos permanecen sin procesar, los comandos independientes de inventario del repositorio pueden compactarse y las secuencias mixtas de comandos no seguras permanecen sin procesar.
- Sigue siendo opcional: desactivar el plugin si se desea una salida literal en todos los casos.

## Verificar que funciona

1. Activar el plugin.
2. Iniciar una sesión que pueda llamar a `exec`.
3. Ejecutar un comando ruidoso como `git status`.
4. Comprobar que el resultado devuelto por la herramienta sea más breve y esté mejor estructurado que la salida sin procesar del shell.

## Desactivar el plugin

```bash
openclaw config set plugins.entries.tokenjuice.enabled false
```

O bien:

```bash
openclaw plugins disable tokenjuice
```

## Temas relacionados

- [Herramienta de ejecución](/es/tools/exec)
- [Niveles de razonamiento](/es/tools/thinking)
- [Motor de contexto](/es/concepts/context-engine)
