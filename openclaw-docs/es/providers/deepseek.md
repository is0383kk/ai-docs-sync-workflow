---
read_when:
    - Quieres usar DeepSeek con OpenClaw
    - Se necesita la variable de entorno de la clave de API o la opción de autenticación de la CLI
summary: Configuración de DeepSeek (autenticación + selección de modelo)
title: DeepSeek
x-i18n:
    generated_at: "2026-07-26T05:25:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e074756d593205d7d05f499da93b9bd3c63acdce7092b42fb5562023577925
    source_path: providers/deepseek.md
    workflow: 16
---

[DeepSeek](https://www.deepseek.com) proporciona potentes modelos de IA con una API compatible con OpenAI.

| Propiedad | Valor                      |
| -------- | -------------------------- |
| Proveedor | `deepseek`                 |
| Autenticación     | `DEEPSEEK_API_KEY`         |
| API      | Compatible con OpenAI          |
| URL base | `https://api.deepseek.com` |

## Instalar el plugin

Instale el plugin oficial y, a continuación, reinicie el Gateway:

```bash
openclaw plugins install @openclaw/deepseek-provider
openclaw gateway restart
```

## Primeros pasos

<Steps>
  <Step title="Obtener la clave de API">
    Cree una clave de API en [platform.deepseek.com](https://platform.deepseek.com/api_keys).
  </Step>
  <Step title="Ejecutar la incorporación">
    ```bash
    openclaw onboard --auth-choice deepseek-api-key
    ```

    Solicita la clave de API y establece `deepseek/deepseek-v4-flash` como modelo predeterminado.

  </Step>
  <Step title="Verificar que los modelos estén disponibles">
    ```bash
    openclaw models list --provider deepseek
    ```

    Para inspeccionar el catálogo estático del plugin sin un Gateway en ejecución:

    ```bash
    openclaw models list --all --provider deepseek
    ```

  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Configuración no interactiva">
    Para instalaciones mediante scripts o sin interfaz gráfica, pase todas las opciones directamente:

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice deepseek-api-key \
      --deepseek-api-key "$DEEPSEEK_API_KEY" \
      --skip-health \
      --accept-risk
    ```

  </Accordion>
</AccordionGroup>

<Warning>
Si el Gateway se ejecuta como un demonio (launchd/systemd), asegúrese de que `DEEPSEEK_API_KEY` esté
disponible para ese proceso (por ejemplo, en `~/.openclaw/.env` o mediante
`env.shellEnv`).
</Warning>

## Catálogo integrado

| Referencia del modelo                    | Nombre              | Entrada | Contexto   | Salida máxima | Notas                                               |
| ---------------------------- | ----------------- | ----- | --------- | ---------- | --------------------------------------------------- |
| `deepseek/deepseek-v4-flash` | DeepSeek V4 Flash | texto  | 1,000,000 | 384,000    | Modelo predeterminado; interfaz V4 compatible con razonamiento          |
| `deepseek/deepseek-v4-pro`   | DeepSeek V4 Pro   | texto  | 1,000,000 | 384,000    | Interfaz V4 compatible con razonamiento                         |
| `deepseek/deepseek-chat`     | DeepSeek Chat     | texto  | 1,000,000 | 384,000    | Nombre de compatibilidad obsoleto para V4 Flash sin razonamiento |
| `deepseek/deepseek-reasoner` | DeepSeek Reasoner | texto  | 1,000,000 | 384,000    | Nombre de compatibilidad obsoleto para V4 Flash con razonamiento     |

<Warning>
DeepSeek retirará `deepseek-chat` y `deepseek-reasoner` el 24 de julio de 2026
a las 15:59 UTC. Actualmente se dirigen a DeepSeek V4 Flash en modo sin razonamiento y
con razonamiento, respectivamente. Cambie las referencias de modelo configuradas a
`deepseek/deepseek-v4-flash` o `deepseek/deepseek-v4-pro` antes de la fecha límite.
</Warning>

Las estimaciones de costes locales de OpenClaw siguen las tarifas publicadas por DeepSeek para
aciertos de caché, fallos de caché y salida. DeepSeek puede cambiar esas tarifas; su
página [Modelos y precios](https://api-docs.deepseek.com/quick_start/pricing/) es
la fuente oficial para la facturación.

<Tip>
Los modelos V4 admiten el control `thinking` de DeepSeek. OpenClaw también reproduce
`reasoning_content` de DeepSeek en los turnos posteriores para que puedan continuar las sesiones
de razonamiento con llamadas a herramientas.
Use `/think xhigh` o `/think max` con los modelos DeepSeek V4 para solicitar el
`reasoning_effort` máximo de DeepSeek; ambos se asignan a `"max"`.
</Tip>

## Razonamiento y herramientas

Las sesiones de razonamiento de DeepSeek V4 requieren que los mensajes reproducidos del asistente procedentes de un
turno con razonamiento habilitado incluyan `reasoning_content` en las solicitudes posteriores.
El plugin de DeepSeek para OpenClaw completa ese campo automáticamente, por lo que el uso normal
de herramientas en varios turnos funciona con `deepseek/deepseek-v4-flash` y
`deepseek/deepseek-v4-pro`, incluso cuando el historial procede de otro
proveedor compatible con OpenAI (sin `reasoning_content` nativo) o de un mensaje
simple del asistente. No se requiere `/new` después de cambiar de proveedor durante una sesión.

Cuando el razonamiento está deshabilitado (incluida la selección **None** de la interfaz), OpenClaw
envía `thinking: { type: "disabled" }` y elimina el `reasoning_content` reproducido
del historial saliente, lo que mantiene la sesión en la ruta de DeepSeek sin razonamiento.

Use `deepseek/deepseek-v4-flash` para la ruta rápida predeterminada. Use
`deepseek/deepseek-v4-pro` para el modelo más potente cuando pueda aceptar un mayor
coste o latencia.

## Pruebas en vivo

Para ejecutar únicamente las comprobaciones directas de los modelos DeepSeek V4 de la suite moderna de pruebas en vivo de modelos:

```bash
OPENCLAW_LIVE_PROVIDERS=deepseek \
OPENCLAW_LIVE_MODELS="deepseek/deepseek-v4-flash,deepseek/deepseek-v4-pro" \
pnpm test:live src/agents/models.profiles.live.test.ts
```

Verifica que ambos modelos V4 completen la ejecución y que los turnos posteriores de razonamiento y uso de herramientas
conserven la carga útil reproducida que requiere DeepSeek.

## Ejemplo de configuración

```json5
{
  env: { DEEPSEEK_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "deepseek/deepseek-v4-flash" },
    },
  },
}
```

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Selección de modelos" href="/es/concepts/model-providers" icon="layers">
    Elección de proveedores, referencias de modelos y comportamiento de conmutación por error.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/configuration-reference" icon="gear">
    Referencia completa de configuración para agentes, modelos y proveedores.
  </Card>
</CardGroup>
