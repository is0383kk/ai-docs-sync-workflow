---
read_when:
    - Quieres usar Cloudflare AI Gateway con OpenClaw
    - Necesita el ID de la cuenta, el ID del Gateway o la variable de entorno de la clave de API
summary: Configuración de Cloudflare AI Gateway (autenticación + selección de modelo)
title: Gateway de IA de Cloudflare
x-i18n:
    generated_at: "2026-07-26T05:53:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 02c7785616e7aee645bb3fc41ef6a3585e1f2f9d886fab1a06231e497effd045
    source_path: providers/cloudflare-ai-gateway.md
    workflow: 16
---

[Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) se sitúa delante de las API de los proveedores y añade analíticas, almacenamiento en caché y controles. Para Anthropic, OpenClaw utiliza la API Messages de Anthropic mediante el endpoint de Gateway.

| Propiedad     | Valor                                                                                    |
| ------------- | ---------------------------------------------------------------------------------------- |
| Proveedor     | `cloudflare-ai-gateway`                                                                  |
| Plugin        | paquete externo oficial (`@openclaw/cloudflare-ai-gateway-provider`)                   |
| URL base      | `https://gateway.ai.cloudflare.com/v1/<account_id>/<gateway_id>/anthropic`               |
| Modelo predeterminado | `cloudflare-ai-gateway/claude-sonnet-4-6`                                                |
| Clave de API  | `CLOUDFLARE_AI_GATEWAY_API_KEY` (la clave de API del proveedor para las solicitudes mediante el Gateway) |

<Note>
Para los modelos de Anthropic enrutados mediante Cloudflare AI Gateway, utilice su **clave de API de Anthropic** como clave del proveedor.
</Note>

Cuando el razonamiento está habilitado para los modelos de Anthropic Messages, OpenClaw elimina los turnos finales
de prellenado del asistente antes de enviar la carga útil mediante Cloudflare AI Gateway.
Anthropic rechaza el prellenado de respuestas con razonamiento extendido, mientras que el prellenado
ordinario sin razonamiento sigue estando disponible.

## Instalar el Plugin

Instale el Plugin oficial y, a continuación, reinicie el Gateway:

```bash
openclaw plugins install @openclaw/cloudflare-ai-gateway-provider
openclaw gateway restart
```

## Primeros pasos

<Steps>
  <Step title="Configurar la clave de API del proveedor y los datos del Gateway">
    Ejecute la incorporación y elija la opción de autenticación de Cloudflare AI Gateway:

    ```bash
    openclaw onboard --auth-choice cloudflare-ai-gateway-api-key
    ```

    Se solicitarán el ID de la cuenta, el ID del gateway y la clave de API.

  </Step>
  <Step title="Configurar un modelo predeterminado">
    Añada el modelo a la configuración de OpenClaw:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "cloudflare-ai-gateway/claude-sonnet-4-6" },
        },
      },
    }
    ```

  </Step>
  <Step title="Verificar que el modelo esté disponible">
    ```bash
    openclaw models list --provider cloudflare-ai-gateway
    ```
  </Step>
</Steps>

## Ejemplo no interactivo

Para configuraciones automatizadas o de CI, pase todos los valores mediante la línea de comandos:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cloudflare-ai-gateway-api-key \
  --cloudflare-ai-gateway-account-id "your-account-id" \
  --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
  --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY"
```

## Configuración avanzada

<AccordionGroup>
  <Accordion title="Gateways autenticados">
    Si ha habilitado la autenticación del Gateway en Cloudflare, añada la cabecera `cf-aig-authorization`. Esto se requiere **además de** la clave de API del proveedor.

    ```json5
    {
      models: {
        providers: {
          "cloudflare-ai-gateway": {
            headers: {
              "cf-aig-authorization": "Bearer <cloudflare-ai-gateway-token>",
            },
          },
        },
      },
    }
    ```

    <Tip>
    La cabecera `cf-aig-authorization` autentica con el propio Cloudflare Gateway, mientras que la clave de API del proveedor (por ejemplo, la clave de Anthropic) autentica con el proveedor ascendente.
    </Tip>

  </Accordion>

  <Accordion title="Nota sobre el entorno">
    Si el Gateway se ejecuta como daemon (launchd/systemd), asegúrese de que `CLOUDFLARE_AI_GATEWAY_API_KEY` esté disponible para ese proceso.

    <Warning>
    Una clave exportada únicamente en un shell interactivo no servirá para un daemon de launchd/systemd, a menos que ese entorno también se importe allí. Configure la clave en `~/.openclaw/.env` o mediante `env.shellEnv` para garantizar que el proceso del Gateway pueda leerla.
    </Warning>

  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Selección de modelos" href="/es/concepts/model-providers" icon="layers">
    Elección de proveedores, referencias de modelos y comportamiento de conmutación por error.
  </Card>
  <Card title="Solución de problemas" href="/es/help/troubleshooting" icon="wrench">
    Solución general de problemas y preguntas frecuentes.
  </Card>
</CardGroup>
