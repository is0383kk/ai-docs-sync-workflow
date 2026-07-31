---
read_when:
    - Se desea poner en cola un evento del sistema sin crear una tarea cron
    - Es necesario habilitar o deshabilitar los heartbeats
    - Se desea inspeccionar las entradas de presencia del sistema
summary: Referencia de la CLI para `openclaw system` (eventos del sistema, Heartbeat, presencia)
title: Sistema
x-i18n:
    generated_at: "2026-07-26T05:09:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aaca206d8b463fd33f9e3cb21382bbf36469e9daa2706d8a9e2c7fab14b76e7a
    source_path: cli/system.md
    workflow: 16
---

# `openclaw system`

Herramientas auxiliares a nivel del sistema para el Gateway: poner en cola eventos del sistema, controlar
los heartbeats y consultar la presencia.

Todos los subcomandos de `system` usan RPC del Gateway y aceptan las opciones compartidas del cliente:

| Opción            | Valor predeterminado                 | Descripción                                                                                                                                                                                            |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--url <url>`     | `gateway.remote.url` cuando está configurado | URL de WebSocket del Gateway.                                                                                                                                                                          |
| `--token <token>` | ninguno                              | Token del Gateway (si es necesario).                                                                                                                                                                   |
| `--timeout <ms>`  | `30000`                              | Tiempo de espera de RPC en milisegundos.                                                                                                                                                               |
| `--expect-final`  | desactivado                          | Esperar la respuesta final (agente).                                                                                                                                                                   |
| `--json`          | desactivado                          | Generar JSON. `heartbeat last/enable/disable` y `system presence` siempre imprimen la carga útil JSON sin procesar de RPC independientemente de esta opción; `system event` la usa para alternar entre JSON y una línea simple de `ok`. |

## Comandos comunes

```bash
openclaw system event --text "Comprobar si hay seguimientos urgentes" --mode now
openclaw system event --text "Comprobar si hay seguimientos urgentes" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

De forma predeterminada, pone en cola un evento del sistema en la sesión **principal**. El siguiente
heartbeat lo inserta como una línea de `System:` en el prompt. Use `--mode now` para
activar el heartbeat de inmediato; `next-heartbeat` (valor predeterminado) espera al
siguiente ciclo programado.

Pase `--session-key` para dirigirse a una sesión específica, por ejemplo, para retransmitir la
finalización de una tarea asíncrona al canal que la inició.

<Note>
**Excepción de temporización con `--session-key`:** cuando se proporciona `--session-key`,
`--mode next-heartbeat` se convierte en una activación dirigida inmediata en lugar de
esperar al siguiente ciclo programado. Las activaciones dirigidas usan la intención de heartbeat
`immediate`, por lo que omiten la comprobación del ejecutor que determina que aún no corresponde ejecutarlo, la cual, de otro modo,
aplazaría (y, en la práctica, descartaría) una activación con intención `event`. Si desea una
entrega retrasada, omita `--session-key` para que el evento llegue a la sesión principal y
se entregue con el siguiente heartbeat habitual.
</Note>

Opciones:

- `--text <text>`: texto obligatorio del evento del sistema.
- `--mode <mode>`: `now` o `next-heartbeat` (valor predeterminado).
- `--session-key <sessionKey>`: opcional; se dirige a una sesión específica del agente
  en lugar de a la sesión principal del agente. Las claves que no pertenecen al
  agente resuelto recurren a la sesión principal del agente.

## `system heartbeat last|enable|disable`

- `last`: muestra el último evento de heartbeat.
- `enable`: vuelve a activar los heartbeats (use esta opción si estaban desactivados).
- `disable`: pausa los heartbeats.

## `system presence`

Enumera las entradas actuales de presencia del sistema que conoce el Gateway (nodos,
instancias y líneas de estado similares).

## Notas

- Requiere un Gateway en ejecución al que se pueda acceder mediante la configuración actual (local o
  remota).
- Los eventos del sistema son efímeros y no se conservan entre reinicios.

## Relacionado

- [Referencia de la CLI](/es/cli)
