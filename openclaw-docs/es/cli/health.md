---
read_when:
    - Se desea comprobar rápidamente el estado del Gateway en ejecución
summary: Referencia de la CLI para `openclaw health` (instantánea del estado del Gateway mediante RPC)
title: Salud
x-i18n:
    generated_at: "2026-07-26T05:06:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51cc0e3dd61af3e6fa460dd646bfa1c3e5bd1a52da860eac26c12101151d081d
    source_path: cli/health.md
    workflow: 16
---

# `openclaw health`

Obtiene una instantánea del estado del Gateway en ejecución mediante RPC por WebSocket (sin conexiones directas de canal desde la CLI).

## Opciones

| Indicador        | Valor predeterminado | Descripción                                                                                                        |
| ---------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `--json`         | `false` | Imprime JSON legible por máquina en lugar de texto.                                                                |
| `--timeout <ms>` | `10000` | Tiempo de espera de conexión en milisegundos.                                                                      |
| `--verbose`      | `false` | Fuerza una comprobación en vivo y amplía la salida para todas las cuentas y los agentes configurados.              |
| `--debug`        | `false` | Alias de `--verbose`.                                                                                       |

Ejemplos:

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

## Comportamiento

- Sin `--verbose`, el Gateway puede devolver una instantánea almacenada en caché (actualizada durante un máximo de 60 segundos y sin cambios respecto al estado del entorno de ejecución de los canales en vivo) y actualizarla en segundo plano para la siguiente solicitud.
- `--verbose` fuerza una comprobación en vivo (comprobaciones de cuentas por canal), muestra los detalles de conexión del Gateway y amplía la salida legible para personas a todas las cuentas y los agentes configurados, en lugar de limitarse al agente predeterminado.
- `--json` siempre devuelve la instantánea completa: canales, comprobaciones por cuenta, estado de carga de los plugins, estado de cuarentena del motor de contexto, estado de la caché de precios de modelos, estado del bucle de eventos, mensajes fallidos de la cola de entrega y almacenes de sesiones por agente.
- Cuando los envíos salientes o los eventos entrantes de los canales se envían a la cola de mensajes fallidos, la salida de texto informa de sus cantidades y de la antigüedad del fallo más antiguo. Las cantidades de eventos entrantes se agrupan por cuenta de canal; se pueden inspeccionar o recuperar eventos individuales con [`openclaw channels dead-letters`](/es/cli/channels#inbound-dead-letters).

## Relacionado

- [Referencia de la CLI](/es/cli)
- [`openclaw status`](/es/cli/status) — diagnóstico local y comprobaciones de canales sin una instantánea completa del estado
- [Estado del Gateway](/es/gateway/health)
