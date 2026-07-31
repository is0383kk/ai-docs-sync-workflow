---
read_when:
    - Quieres usar Featherless AI con OpenClaw
    - Necesitas la variable de entorno de la clave de API de Featherless o el formato de referencia del modelo
summary: Configuración de Featherless AI, selección de modelos y llamadas a herramientas
title: Featherless AI
x-i18n:
    generated_at: "2026-07-26T04:49:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9112f7e65b4089bf96933c632d0b62f7fb87d42998d985ca85eb92dc392636b6
    source_path: providers/featherless.md
    workflow: 16
---

[Featherless AI](https://featherless.ai) ofrece modelos abiertos mediante una
API compatible con OpenAI. OpenClaw instala Featherless como Plugin de proveedor
externo oficial y mantiene reducido el catálogo integrado, a la vez que acepta
los identificadores exactos de modelos de Featherless durante la ejecución.

| Propiedad                | Valor                                    |
| ------------------------ | ---------------------------------------- |
| Id. del proveedor        | `featherless`                            |
| Paquete                  | `@openclaw/featherless-provider`         |
| Variable de entorno de autenticación | `FEATHERLESS_API_KEY`                    |
| Indicador de incorporación | `--auth-choice featherless-api-key`      |
| Indicador directo de la CLI | `--featherless-api-key <key>`            |
| API                      | Compatible con OpenAI (`openai-completions`) |
| URL base                 | `https://api.featherless.ai/v1`          |
| Modelo predeterminado    | `featherless/Qwen/Qwen3-32B`             |

## Configuración

Instale el Plugin y reinicie el Gateway:

```bash
openclaw plugins install @openclaw/featherless-provider
openclaw gateway restart
```

Ejecute la incorporación:

```bash
openclaw onboard --auth-choice featherless-api-key
```

Para una configuración no interactiva:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice featherless-api-key \
  --featherless-api-key "$FEATHERLESS_API_KEY"
```

O exponga la clave al proceso del Gateway:

```bash
export FEATHERLESS_API_KEY="<your-featherless-api-key>" # pragma: allowlist secret
```

Verifique el proveedor:

```bash
openclaw models list --provider featherless
```

## Modelo predeterminado

El Plugin usa `Qwen/Qwen3-32B` como valor predeterminado de configuración porque Featherless
documenta el uso nativo de herramientas para la familia Qwen 3. OpenClaw configura
su ventana de contexto de 32,768 tokens, un límite de salida conservador de 4,096 tokens y
los controles de razonamiento de la plantilla de chat de Qwen.

Los campos de coste del catálogo son cero porque Featherless admite varios modos
de facturación y OpenClaw no incorpora tarifas específicas de la cuenta para el plan
ni para el precio por solicitud.

## Otros modelos de Featherless

Use el identificador exacto del modelo de Featherless después del prefijo de proveedor `featherless/`:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "featherless/moonshotai/Kimi-K2-Instruct",
      },
    },
  },
}
```

OpenClaw no copia deliberadamente el índice público completo de modelos de Featherless
en el selector. El índice es grande y no ofrece suficientes metadatos estructurados
sobre capacidades para clasificar de forma segura cada modelo de texto, visión,
incrustación y razonamiento. Por lo tanto, los identificadores desconocidos se resuelven
con valores predeterminados conservadores, solo de texto y sin razonamiento: una ventana
de contexto de 4,096 tokens y un límite de salida de 1,024 tokens.

Añada una entrada explícita de modelo del proveedor cuando un modelo necesite metadatos diferentes:

```json5
{
  models: {
    mode: "merge",
    providers: {
      featherless: {
        baseUrl: "https://api.featherless.ai/v1",
        apiKey: "${FEATHERLESS_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-3-27b-it",
            name: "Gemma 3 27B",
            input: ["text", "image"],
            reasoning: false,
            contextWindow: 32768,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

Consulte el catálogo de modelos de Featherless para comprobar la disponibilidad actual
de los modelos y las etiquetas de capacidad antes de añadir metadatos personalizados.

## Solución de problemas

- `401` o `403`: confirme que `FEATHERLESS_API_KEY` sea visible para el proceso
  del Gateway, o vuelva a ejecutar la incorporación.
- Modelo desconocido: use el identificador exacto, con distinción entre mayúsculas y minúsculas, de Featherless después del
  prefijo `featherless/`.
- Llamadas a herramientas devueltas como texto: elija una familia de modelos que Featherless documente para
  llamadas nativas a funciones, como Qwen 3.
- El Gateway gestionado no puede ver la clave: colóquela en `~/.openclaw/.env` o en otra
  fuente de entorno cargada por el servicio y, después, reinicie el Gateway.

## Relacionado

- [Proveedores de modelos](/es/concepts/model-providers)
- [Todos los proveedores](/es/providers/index)
- [Modos de razonamiento](/es/tools/thinking)
