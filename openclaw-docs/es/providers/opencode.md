---
read_when:
    - Quieres acceder a modelos alojados en OpenCode
    - Quieres elegir entre los catálogos Zen y Go
summary: Usar los catálogos Zen y Go de OpenCode con OpenClaw
title: OpenCode
x-i18n:
    generated_at: "2026-07-26T05:54:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de287eb8a349f26c265f95b8b1de3af4035aa2bdc3501c7279f714d297bb8b9b
    source_path: providers/opencode.md
    workflow: 16
---

OpenCode expone dos catálogos alojados en OpenClaw:

| Catálogo | Prefijo            | Proveedor de runtime |
| -------- | ------------------ | -------------------- |
| **Zen**  | `opencode/...` | `opencode`   |
| **Go**   | `opencode-go/...` | `opencode-go`   |

Ambos catálogos comparten una clave de API de OpenCode (`OPENCODE_API_KEY`, alias
`OPENCODE_ZEN_API_KEY`). OpenClaw mantiene separados los identificadores de los proveedores de runtime para que
el enrutamiento ascendente por modelo siga siendo correcto, pero la incorporación y la documentación los tratan como
una única configuración de OpenCode.

## Primeros pasos

<Tabs>
  <Tab title="Catálogo Zen">
    **Ideal para:** el proxy multimodelo seleccionado de OpenCode (Claude, GPT, Gemini, GLM,
    DeepSeek, Kimi, MiniMax, Qwen).

    <Steps>
      <Step title="Ejecutar la incorporación">
        ```bash
        openclaw onboard --auth-choice opencode-zen
        ```

        O pasar la clave directamente:

        ```bash
        openclaw onboard --opencode-zen-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Establecer un modelo Zen como predeterminado">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode/claude-opus-4-6"
        ```
      </Step>
      <Step title="Verificar que los modelos estén disponibles">
        ```bash
        openclaw models list --provider opencode
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Catálogo Go">
    **Ideal para:** la gama de Kimi, GLM, MiniMax, Qwen y DeepSeek alojada en OpenCode.

    <Steps>
      <Step title="Ejecutar la incorporación">
        ```bash
        openclaw onboard --auth-choice opencode-go
        ```

        O pasar la clave directamente:

        ```bash
        openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Establecer un modelo Go como predeterminado">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode-go/kimi-k2.6"
        ```
      </Step>
      <Step title="Verificar que los modelos estén disponibles">
        ```bash
        openclaw models list --provider opencode-go
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Ejemplo de configuración

```json5
{
  env: { OPENCODE_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

## Catálogos integrados

### Zen

| Propiedad            | Valor                                                                                         |
| -------------------- | --------------------------------------------------------------------------------------------- |
| Proveedor de runtime | `opencode`                                                                            |
| Modelos de ejemplo   | `opencode/claude-opus-4-6`, `opencode/gpt-5.5`, `opencode/gemini-3.1-pro`, `opencode/glm-5.2`                |

Ejecutar `openclaw models list --provider opencode` para obtener la lista actual completa, que
también incluye filas del nivel gratuito como `opencode/big-pickle` y
`opencode/deepseek-v4-flash-free`.

### Go

| Propiedad            | Valor                                                         |
| -------------------- | ------------------------------------------------------------- |
| Proveedor de runtime | `opencode-go`                                             |
| Modelos de ejemplo   | `opencode-go/kimi-k2.6`, `opencode-go/glm-5`, `opencode-go/minimax-m2.5`     |

Consultar [OpenCode Go](/es/providers/opencode-go) para ver la tabla completa de modelos Go.

## Configuración avanzada

<AccordionGroup>
  <Accordion title="Alias de claves de API">
    `OPENCODE_ZEN_API_KEY` también se acepta como alias de `OPENCODE_API_KEY`.
  </Accordion>

  <Accordion title="Credenciales compartidas">
    Al introducir una clave de OpenCode durante la configuración, se almacenan credenciales para ambos
    proveedores de runtime. No es necesario incorporar cada catálogo por separado.
  </Accordion>

  <Accordion title="Obtener una clave de API">
    Crear una cuenta de OpenCode y generar una clave de API en
    [opencode.ai/auth](https://opencode.ai/auth). La facturación y la disponibilidad del catálogo
    se administran desde el panel de OpenCode.
  </Accordion>

  <Accordion title="Comportamiento de repetición de Gemini">
    Las referencias de OpenCode respaldadas por Gemini permanecen en la ruta proxy-Gemini, por lo que OpenClaw mantiene
    allí la depuración de firmas de pensamiento de Gemini sin habilitar la validación de repetición nativa de Gemini
    ni las reescrituras de arranque.
  </Accordion>

  <Accordion title="Comportamiento de repetición para modelos distintos de Gemini">
    Las referencias de OpenCode que no son de Gemini mantienen la política mínima de repetición compatible con OpenAI.
  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="OpenCode Go" href="/es/providers/opencode-go" icon="server">
    Referencia completa del catálogo Go.
  </Card>
  <Card title="Selección de modelos" href="/es/concepts/model-providers" icon="layers">
    Selección de proveedores, referencias de modelos y comportamiento de conmutación por error.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/configuration-reference" icon="gear">
    Referencia completa de configuración para agentes, modelos y proveedores.
  </Card>
</CardGroup>
