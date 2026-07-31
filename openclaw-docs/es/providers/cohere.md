---
read_when:
    - Se desea usar Cohere con OpenClaw
    - Se necesita la variable de entorno de la clave de API de Cohere o la opción de autenticación de la CLI
summary: Configuración de Cohere (autenticación + selección del modelo)
title: Cohere
x-i18n:
    generated_at: "2026-07-26T04:50:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fee46bf80609bd5e8211d6be507713f4de178653941effb81ebae48d8bb6528a
    source_path: providers/cohere.md
    workflow: 16
---

[Cohere](https://cohere.com) proporciona inferencia compatible con OpenAI mediante su API de compatibilidad. OpenClaw incluye el proveedor Cohere durante su transición a la externalización y también lo publica como Plugin externo oficial.

| Propiedad                  | Valor                                                      |
| -------------------------- | ---------------------------------------------------------- |
| Id. del proveedor          | `cohere`                                         |
| Plugin                     | incluido durante la transición; paquete externo oficial    |
| Variable de entorno de autenticación | `COHERE_API_KEY`                              |
| Opción de incorporación    | `--auth-choice cohere-api-key`                                         |
| Opción directa de la CLI   | `--cohere-api-key <key>`                                         |
| API                        | compatible con OpenAI (`openai-completions`)                 |
| URL base                   | `https://api.cohere.ai/compatibility/v1`                                         |
| Modelo predeterminado      | `cohere/command-a-plus-05-2026`                                         |
| Ventana de contexto        | 128,000 tokens                                             |

## Catálogo integrado

| Referencia del modelo                  | Entrada        | Contexto | Salida máxima | Notas                                                   |
| -------------------------------------- | -------------- | -------- | ------------- | ------------------------------------------------------- |
| `cohere/command-a-plus-05-2026`                     | texto, imagen  | 128,000  | 64,000        | Predeterminado; modelo principal agéntico y de razonamiento |
| `cohere/command-a-03-2025`                     | texto          | 256,000  | 8,000         | Modelo Command A anterior                               |
| `cohere/command-a-reasoning-08-2025`                     | texto          | 256,000  | 32,000        | Razonamiento agéntico y uso de herramientas             |
| `cohere/command-a-vision-07-2025`                     | texto, imagen  | 128,000  | 8,000         | Análisis visual y de documentos; sin uso de herramientas |
| `cohere/north-mini-code-1-0`                     | texto, imagen  | 256,000  | 64,000        | Programación agéntica; razonamiento; límites gratuitos  |

Los modelos Cohere con capacidad de razonamiento admiten dos modos de razonamiento de la API de compatibilidad. OpenClaw asigna **desactivado** a `none` y todos los niveles de pensamiento habilitados a `high`. Command A Vision no admite el uso de herramientas, por lo que OpenClaw mantiene deshabilitadas las herramientas del agente para ese modelo.

## Primeros pasos

1. Cohere se incluye con los paquetes actuales de OpenClaw. Si falta, instale el paquete externo y reinicie el Gateway:

```bash
openclaw plugins install @openclaw/cohere-provider
openclaw gateway restart
```

2. Cree una clave de API de Cohere.
3. Ejecute la incorporación:

```bash
openclaw onboard --non-interactive \
  --auth-choice cohere-api-key \
  --cohere-api-key "$COHERE_API_KEY"
```

4. Confirme que el catálogo esté disponible:

```bash
openclaw models list --provider cohere
```

La incorporación solo establece Cohere como modelo principal cuando aún no hay ningún modelo principal configurado.

## Configuración solo mediante el entorno

Haga que `COHERE_API_KEY` esté disponible para el proceso del Gateway y, a continuación, seleccione el modelo Cohere:

```json5
{
  agents: {
    defaults: {
      model: { primary: "cohere/command-a-plus-05-2026" },
    },
  },
}
```

<Note>
Si el Gateway se ejecuta como demonio o en Docker, establezca `COHERE_API_KEY` para ese servicio. Exportarla únicamente en un shell interactivo no hace que esté disponible para un Gateway que ya se está ejecutando.
</Note>

## Temas relacionados

- [Proveedores de modelos](/es/concepts/model-providers)
- [CLI de modelos](/es/cli/models)
- [Directorio de proveedores](/es/providers/index)
