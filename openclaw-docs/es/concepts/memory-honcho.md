---
read_when:
    - Se necesita memoria persistente que funcione entre sesiones y canales
    - Quieres recuperación de información y modelado de usuarios con tecnología de IA
summary: Memoria entre sesiones nativa de IA mediante el plugin Honcho
title: Memoria de Honcho
x-i18n:
    generated_at: "2026-07-26T04:36:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fadcf6d8e2505ab4fe6a81340695b7c8fee49c3cb4889665af13389941619117
    source_path: concepts/memory-honcho.md
    workflow: 16
---

[Honcho](https://honcho.dev) añade memoria nativa para IA a OpenClaw mediante un
plugin externo. Conserva las conversaciones en un servicio dedicado y crea
modelos del usuario y del agente con el tiempo, lo que proporciona al agente
contexto entre sesiones que va más allá de los archivos Markdown del espacio de trabajo.

## Qué proporciona

- **Memoria entre sesiones** - las conversaciones se conservan después de cada turno, por lo que
  el contexto se mantiene tras reinicios de sesión, la compactación y los cambios de canal.
- **Modelado de usuarios** - Honcho mantiene un perfil para cada usuario (preferencias,
  datos, estilo de comunicación) y para el agente (personalidad, comportamientos
  aprendidos).
- **Búsqueda semántica** - busca entre observaciones de conversaciones anteriores, no
  solo en la sesión actual.
- **Conocimiento de múltiples agentes** - los agentes principales realizan automáticamente el seguimiento de los
  subagentes creados, y se añaden como observadores en las sesiones secundarias.

## Herramientas disponibles

Honcho registra herramientas que el agente puede utilizar durante la conversación:

**Recuperación de datos (rápida, sin llamada al LLM):**

| Herramienta                 | Qué hace                                                |
| --------------------------- | ------------------------------------------------------ |
| `honcho_context`            | Representación completa del usuario entre sesiones      |
| `honcho_search_conclusions` | Búsqueda semántica entre conclusiones almacenadas       |
| `honcho_search_messages`    | Busca mensajes entre sesiones (filtra por remitente, fecha) |
| `honcho_session`            | Historial y resumen de la sesión actual                 |

**Preguntas y respuestas (mediante LLM):**

| Herramienta  | Qué hace                                                                   |
| ------------ | ------------------------------------------------------------------------- |
| `honcho_ask` | Pregunta sobre el usuario. `depth='quick'` para datos, `'thorough'` para síntesis |

## Primeros pasos

Instale el plugin y ejecute la configuración:

```bash
openclaw plugins install @honcho-ai/openclaw-honcho
openclaw honcho setup
openclaw gateway --force
```

El comando de configuración solicita las credenciales de API, escribe la configuración y
permite migrar opcionalmente los archivos de memoria existentes del espacio de trabajo.

<Info>
Honcho puede ejecutarse completamente de forma local (autoalojado) o mediante la API administrada en
`api.honcho.dev`. La opción autoalojada no requiere dependencias
externas.
</Info>

## Configuración

Los ajustes se encuentran en `plugins.entries["openclaw-honcho"].config`:

```json5
{
  plugins: {
    entries: {
      "openclaw-honcho": {
        config: {
          apiKey: "your-api-key", // omitir para autoalojamiento
          workspaceId: "openclaw", // aislamiento de memoria
          baseUrl: "https://api.honcho.dev",
        },
      },
    },
  },
}
```

Para las instancias autoalojadas, dirija `baseUrl` al servidor local (por ejemplo,
`http://localhost:8000`) y omita la clave de API.

## Migración de memoria existente

Si existen archivos de memoria en el espacio de trabajo (`USER.md`, `MEMORY.md`,
`IDENTITY.md`, `memory/`, `canvas/`), `openclaw honcho setup` los detecta y
ofrece migrarlos.

<Info>
La migración no es destructiva: los archivos se cargan en Honcho. Los originales
nunca se eliminan ni se mueven.
</Info>

## Cómo funciona

Después de cada turno de la IA, la conversación se conserva en Honcho. Se observan tanto los
mensajes del usuario como los del agente, lo que permite a Honcho crear y perfeccionar sus modelos con el
tiempo.

Durante la conversación, las herramientas de Honcho consultan el servicio durante el enlace de plugin
`before_prompt_build` de OpenClaw e insertan el contexto pertinente antes de que el modelo
reciba el mensaje.

## Honcho frente a la memoria integrada

|                   | Integrada / QMD                    | Honcho                                  |
| ----------------- | ---------------------------------- | --------------------------------------- |
| **Almacenamiento** | Archivos Markdown del espacio de trabajo | Servicio dedicado (local o alojado) |
| **Entre sesiones** | Mediante archivos de memoria       | Automático e integrado                   |
| **Modelado de usuarios** | Manual (escribir en MEMORY.md) | Perfiles automáticos                 |
| **Búsqueda**       | Vectores + palabras clave (híbrida) | Semántica entre observaciones           |
| **Múltiples agentes** | Sin seguimiento                 | Conocimiento de relaciones principal/secundario |
| **Dependencias**   | Ninguna (integrada) o binario QMD  | Instalación del plugin                   |

Honcho y el sistema de memoria integrado pueden funcionar conjuntamente. Cuando QMD está
configurado, se habilitan herramientas adicionales para buscar en archivos Markdown locales
junto con la memoria entre sesiones de Honcho.

## Comandos de la CLI

```bash
openclaw honcho setup                        # Configurar la clave de API y migrar archivos
openclaw honcho status                       # Comprobar el estado de la conexión
openclaw honcho ask <question>               # Consultar a Honcho sobre el usuario
openclaw honcho search <query> [-k N] [-d D] # Búsqueda semántica en la memoria
```

## Lecturas adicionales

- [Código fuente del plugin](https://github.com/plastic-labs/openclaw-honcho)
- [Documentación de Honcho](https://docs.honcho.dev)
- [Guía de integración de Honcho con OpenClaw](https://docs.honcho.dev/v3/guides/integrations/openclaw)

## Contenido relacionado

- [Descripción general de la memoria](/es/concepts/memory)
- [Motor de memoria integrado](/es/concepts/memory-builtin)
- [Motor de memoria QMD](/es/concepts/memory-qmd)
- [Motores de contexto](/es/concepts/context-engine)
