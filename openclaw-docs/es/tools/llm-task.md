---
read_when:
    - Quiere un paso de LLM que solo genere JSON dentro de los flujos de trabajo
    - Necesita una salida del LLM validada mediante un esquema para la automatización
summary: Tareas de LLM exclusivamente en JSON para flujos de trabajo (herramienta de Plugin opcional)
title: Tarea del LLM
x-i18n:
    generated_at: "2026-07-26T05:24:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 78ea533f43546fbdd66c7f7138b8dea0b12b02d38925689324b390a12d0c4c5a
    source_path: tools/llm-task.md
    workflow: 16
---

`llm-task` es una **herramienta de plugin opcional** incluida que ejecuta una única
llamada a un LLM solo con JSON y devuelve una salida estructurada, validada opcionalmente con un
esquema JSON. Proporciona a motores de flujo de trabajo como Lobster un paso de LLM sin código
personalizado de OpenClaw para cada flujo de trabajo.

## Habilitar

1. Habilite el plugin:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  }
}
```

2. Permita la herramienta:

```json
{
  "tools": {
    "alsoAllow": ["llm-task"]
  }
}
```

`alsoAllow` añade `llm-task` al perfil de herramientas activo sin
restringir otras herramientas principales. Utilice `tools.allow` en su lugar solo si desea un modo
de lista de permitidos restrictivo.

## Configuración (opcional)

```json
{
  "plugins": {
    "entries": {
      "llm-task": {
        "enabled": true,
        "config": {
          "defaultProvider": "openai",
          "defaultModel": "gpt-5.6-sol",
          "defaultAuthProfileId": "main",
          "allowedModels": ["openai/gpt-5.6-sol"],
          "maxTokens": 800,
          "timeoutMs": 30000
        }
      }
    }
  }
}
```

`allowedModels` es una lista de permitidos de cadenas `provider/model`; se rechazan las solicitudes de cualquier
otro modelo. Todas las demás claves son valores alternativos por llamada que se utilizan cuando la
llamada a la herramienta omite ese parámetro.

## Parámetros de la herramienta

| Parámetro       | Tipo   | Notas                                                                                                                                         |
| --------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt`        | cadena | Obligatorio. Instrucción de la tarea para el LLM.                                                                                                       |
| `input`         | cualquiera    | Carga útil opcional; se serializa a JSON y se añade al prompt.                                                                              |
| `schema`        | objeto | Esquema JSON opcional con el que debe validarse la salida analizada.                                                                                 |
| `provider`      | cadena | Sobrescribe `defaultProvider` / el proveedor predeterminado del agente.                                                                                   |
| `model`         | cadena | Sobrescribe `defaultModel`; acepta identificadores de modelo simples, alias o una referencia `provider/model` (los prefijos de proveedor duplicados se eliminan automáticamente). |
| `thinking`      | cadena | Nivel de razonamiento (p. ej., `low`, `medium`); debe ser uno de los admitidos por el modelo resuelto.                                                          |
| `authProfileId` | cadena | Sobrescribe `defaultAuthProfileId`.                                                                                                             |
| `temperature`   | número | Se aplica en la medida de lo posible; no todos los proveedores lo respetan.                                                                                                      |
| `maxTokens`     | número | Límite máximo orientativo de tokens de salida.                                                                                                             |
| `timeoutMs`     | número | Tiempo de espera de la ejecución; valor predeterminado: `30000`.                                                                                                                 |

## Salida

Devuelve `details.json` (el JSON analizado y validado con el esquema), además de `details.provider`
y `details.model`, que indican lo que se ejecutó realmente.

## Ejemplo: paso de un flujo de trabajo de Lobster

### Limitación importante

El ejemplo siguiente presupone que la **CLI independiente de Lobster** se ejecuta donde
`openclaw.invoke` ya tiene el contexto correcto de URL/autenticación del Gateway.

Para el ejecutor de Lobster **integrado** incluido en OpenClaw, este patrón de CLI
anidada **no es fiable actualmente**:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

Hasta que Lobster integrado disponga de un puente compatible con este flujo, se recomienda usar:

- llamadas directas a la herramienta `llm-task` fuera de Lobster, o
- pasos de Lobster que no dependan de llamadas anidadas a `openclaw.invoke`.

Ejemplo de la CLI independiente de Lobster:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "A partir del correo electrónico de entrada, devuelve la intención y un borrador.",
  "thinking": "low",
  "input": {
    "subject": "Hola",
    "body": "¿Puedes ayudarme?"
  },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

## Notas de seguridad

- **Solo JSON**: se indica al modelo que devuelva únicamente un valor JSON, sin bloques
  de código ni comentarios.
- **Sin herramientas**: la ejecución subyacente tiene las herramientas deshabilitadas, por lo que el modelo no puede realizar
  llamadas externas durante la tarea.
- Trate la salida como no fiable a menos que la valide con `schema`.
- Exija aprobaciones antes de cualquier paso con efectos secundarios (enviar, publicar, ejecutar) que consuma
  esta salida.

## Relacionado

- [Niveles de razonamiento](/es/tools/thinking)
- [Subagentes](/es/tools/subagents)
- [Comandos con barra](/es/tools/slash-commands)
