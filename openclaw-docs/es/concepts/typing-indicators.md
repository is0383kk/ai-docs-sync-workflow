---
read_when:
    - Cambiar el comportamiento o los valores predeterminados del indicador de escritura
summary: Cuándo muestra OpenClaw indicadores de escritura y cómo ajustarlos
title: Indicadores de escritura
x-i18n:
    generated_at: "2026-07-26T05:09:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3c66d61ea7e3e809b8e88ae2eabb9794f0886b629094753716ed02912843ffc
    source_path: concepts/typing-indicators.md
    workflow: 16
---

Los indicadores de escritura se envían al canal de chat mientras una ejecución está activa. Use `agents.defaults.typingMode` para controlar **cuándo** comienza la escritura y `typingIntervalSeconds` para controlar **con qué frecuencia** se actualiza (cadencia de mantenimiento de conexión, 6 segundos de forma predeterminada).

## Valores predeterminados

Cuando `agents.defaults.typingMode` **no está establecido**:

- **Chats directos**: la escritura comienza inmediatamente cuando se inicia el bucle del modelo.
- **Chats grupales con una mención**: la escritura comienza inmediatamente.
- **Chats grupales sin una mención**: la escritura comienza cuando la ejecución admitida tiene actividad visible para el usuario, como actividad de ejecución del entorno o texto de mensaje.
- **Ejecuciones de Heartbeat**: la escritura comienza cuando se inicia la ejecución de Heartbeat, si el destino de Heartbeat resuelto es un chat que admite indicadores de escritura y estos no están deshabilitados.

## Modos

Establezca `agents.defaults.typingMode` en uno de los siguientes valores:

- `never` - ningún indicador de escritura, nunca.
- `instant` - comienza a escribir **en cuanto se inicia el bucle del modelo**, incluso si la ejecución devuelve posteriormente solo el token de respuesta silenciosa.
- `thinking` - comienza a escribir con el **primer delta de razonamiento** o con la ejecución activa del entorno después de que se acepte el turno.
- `message` - comienza a escribir con la **primera actividad de respuesta visible para el usuario**, como una ejecución activa del entorno o un delta de texto no silencioso. Los tokens de respuesta silenciosa como `NO_REPLY` no cuentan como actividad de texto.

Orden de «antelación con la que se activa»: `never` -> `message`/`thinking` -> `instant`.

## Configuración

Establezca el valor predeterminado para el agente:

```json5
{
  agents: {
    defaults: {
      typingMode: "thinking",
      typingIntervalSeconds: 6,
    },
  },
}
```

Anule la política para un agente:

```json5
{
  agents: {
    entries: {
      support: {
        typingMode: "message",
      },
    },
  },
}
```

## Notas

- El modo `message` no se inicia a partir de tokens de respuesta silenciosa, pero la ejecución activa puede seguir mostrando el indicador de escritura antes de que haya texto del asistente disponible.
- `thinking` sigue reaccionando al razonamiento transmitido (`reasoningLevel: "stream"`) y también puede iniciarse a partir de la ejecución activa antes de que lleguen los deltas de razonamiento.
- El indicador de escritura de Heartbeat es una señal de actividad para el destino de entrega resuelto. Comienza al iniciarse la ejecución de Heartbeat, en lugar de seguir los tiempos de transmisión de `message` o `thinking`. Establezca `typingMode: "never"` para deshabilitarlo.
- Los Heartbeats no muestran el indicador de escritura cuando el destino de Heartbeat es `"none"`, cuando no se puede resolver el destino, cuando la entrega por chat está deshabilitada para el Heartbeat o cuando el canal no admite indicadores de escritura.
- `agents.defaults.typingIntervalSeconds` controla la **cadencia de actualización** de todos los agentes, no la hora de inicio. Valor predeterminado: 6 segundos.

## Temas relacionados

<CardGroup cols={2}>
  <Card title="Presencia" href="/es/concepts/presence" icon="signal">
    Cómo realiza el Gateway el seguimiento de los clientes conectados para la página Dispositivos de la interfaz de control y la pestaña Instancias de macOS.
  </Card>
  <Card title="Transmisión y fragmentación" href="/es/concepts/streaming" icon="bars-staggered">
    Comportamiento de la transmisión saliente, límites de los fragmentos y entrega específica de cada canal.
  </Card>
</CardGroup>
