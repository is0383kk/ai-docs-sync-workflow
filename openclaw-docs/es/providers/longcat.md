---
read_when:
    - Quieres usar LongCat-2.0 con OpenClaw
    - Necesita la clave de API de LongCat o los límites del modelo
summary: Configuración de la API de LongCat para LongCat-2.0
title: LongCat
x-i18n:
    generated_at: "2026-07-26T04:49:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c447f9c42e6547a69d2124debcb685c32fe59de29bfc551e18e791d9f280584
    source_path: providers/longcat.md
    workflow: 16
---

[LongCat](https://longcat.ai) proporciona una API alojada para LongCat-2.0, un
modelo de razonamiento diseñado para cargas de trabajo de programación y con agentes. OpenClaw proporciona el
plugin oficial `longcat` para el endpoint compatible con OpenAI de LongCat.

| Propiedad       | Valor                                      |
| --------------- | ------------------------------------------ |
| Proveedor       | `longcat`                         |
| Autenticación   | `LONGCAT_API_KEY`                         |
| API             | Chat Completions compatible con OpenAI     |
| URL base        | `https://api.longcat.chat/openai`                         |
| Modelo          | `longcat/LongCat-2.0`                         |
| Contexto        | 1,048,576 tokens                           |
| Salida máxima   | 131,072 tokens                             |
| Entrada         | Texto                                      |

## Instalar el plugin

Instale el paquete oficial y, a continuación, reinicie el Gateway:

```bash
openclaw plugins install @openclaw/longcat-provider
openclaw gateway restart
```

## Primeros pasos

<Steps>
  <Step title="Crear una clave de API">
    Inicie sesión en la [plataforma de API de LongCat](https://longcat.chat/platform/) y
    cree una clave en la página [API Keys](https://longcat.chat/platform/api_keys).
  </Step>
  <Step title="Ejecutar la incorporación">
    ```bash
    openclaw onboard --auth-choice longcat-api-key
    ```
  </Step>
  <Step title="Verificar el modelo">
    ```bash
    openclaw models list --provider longcat
    ```
  </Step>
</Steps>

La incorporación añade el catálogo alojado y selecciona `longcat/LongCat-2.0` cuando aún no
se ha configurado ningún modelo principal.

### Configuración no interactiva

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice longcat-api-key \
  --longcat-api-key "$LONGCAT_API_KEY"
```

## Comportamiento del razonamiento

LongCat ofrece un control binario del pensamiento. OpenClaw asigna los niveles de pensamiento habilitados
a `thinking: { type: "enabled" }` y `/think off` a
`thinking: { type: "disabled" }`. Actualmente, LongCat no documenta
`reasoning_effort`, por lo que OpenClaw no lo envía.

LongCat devuelve el razonamiento en `reasoning_content`. OpenClaw conserva ese campo
al reproducir turnos de llamadas a herramientas del asistente, de modo que las sesiones de agentes de varios turnos mantengan
la estructura de mensajes esperada por el proveedor.

## Precios

El catálogo integrado utiliza los precios de lista de pago por uso de LongCat en USD por millón de
tokens: $0.75 por entrada sin caché, $0.015 por entrada en caché y $2.95 por salida. LongCat puede
ofrecer descuentos temporales; la [página de precios](https://longcat.chat/platform/docs/Pricing/LongCat-2.0.html)
y los registros de facturación son las fuentes autorizadas.

## LongCat-2.0 autoalojado

El proveedor `longcat` está destinado a la API alojada de LongCat. Para los pesos abiertos en
[Hugging Face](https://huggingface.co/meituan-longcat/LongCat-2.0), sirva el
modelo mediante un entorno de ejecución compatible con OpenAI y utilice en su lugar el proveedor
[vLLM](/es/providers/vllm) o [SGLang](/es/providers/sglang) existente de OpenClaw.

Mantenga el identificador exacto del modelo del entorno de ejecución en el catálogo del proveedor autoalojado;
no enrute una implementación local mediante `longcat/LongCat-2.0`.

## Solución de problemas

<AccordionGroup>
  <Accordion title="La clave funciona en un shell, pero no en el Gateway">
    Los procesos del Gateway administrados por un daemon no heredan todas las variables del shell
    interactivo. Coloque `LONGCAT_API_KEY` en `~/.openclaw/.env`, configúrela mediante
    la incorporación o utilice una referencia de secreto aprobada.
  </Accordion>

  <Accordion title="Las solicitudes fallan con 402 o 429">
    `402` significa que la cuenta no tiene suficiente cuota de tokens. `429` significa que la clave de
    API alcanzó un límite de frecuencia. Compruebe el [uso de LongCat](https://longcat.chat/platform/usage)
    y vuelva a intentar las solicitudes limitadas por frecuencia después del intervalo de espera del proveedor.
  </Accordion>

  <Accordion title="El modelo no aparece">
    Ejecute `openclaw plugins list` y confirme que el plugin `longcat` esté
    habilitado; a continuación, ejecute `openclaw models list --provider longcat`.
  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Proveedores de modelos" href="/es/concepts/model-providers" icon="layers">
    Configuración de proveedores, referencias de modelos y comportamiento de conmutación por error.
  </Card>
  <Card title="Documentación de la API de LongCat" href="https://longcat.chat/platform/docs/" icon="arrow-up-right-from-square">
    Endpoints de la API alojada, autenticación, límites y ejemplos.
  </Card>
  <Card title="Ficha del modelo LongCat-2.0" href="https://huggingface.co/meituan-longcat/LongCat-2.0" icon="arrow-up-right-from-square">
    Arquitectura, orientación para la implementación y detalles del modelo.
  </Card>
  <Card title="Secretos" href="/es/gateway/secrets" icon="key">
    Almacene las credenciales del proveedor sin insertar texto sin formato en la configuración.
  </Card>
</CardGroup>
