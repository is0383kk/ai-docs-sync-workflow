---
read_when:
    - Necesitas seguir en tiempo real los registros del Gateway de forma remota (sin SSH)
    - Se necesitan líneas de registro JSON para las herramientas
summary: Referencia de la CLI para `openclaw logs` (seguir los registros del Gateway mediante RPC)
title: Registros
x-i18n:
    generated_at: "2026-07-26T04:33:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

Sigue los registros de archivo del Gateway mediante RPC. Funciona en modo remoto.

## Opciones

- `--limit <n>`: número máximo de líneas de registro que se devolverán (valor predeterminado: `200`)
- `--max-bytes <n>`: número máximo de bytes que se leerán del archivo de registro (valor predeterminado: `250000`)
- `--follow`: seguir el flujo de registros
- `--interval <ms>`: intervalo de sondeo durante el seguimiento (valor predeterminado: `1000`)
- `--json`: emitir eventos JSON delimitados por líneas
- `--plain`: salida de texto sin formato, sin formato visual
- `--no-color`: desactivar los colores ANSI
- `--local-time`: mostrar las marcas de tiempo en la zona horaria local (valor predeterminado)
- `--utc`: mostrar las marcas de tiempo en UTC

## Opciones compartidas de RPC del Gateway

- `--url <url>`: URL WebSocket del Gateway
- `--token <token>`: token del Gateway
- `--timeout <ms>`: tiempo de espera en ms (valor predeterminado: `30000`)
- `--expect-final`: esperar una respuesta final cuando la llamada al Gateway esté respaldada por un agente

Pasar `--url` omite las credenciales de configuración aplicadas automáticamente; incluya `--token` explícitamente si el Gateway de destino requiere autenticación.

## Ejemplos

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

El perfil raíz seleccionado coincide con el archivo rotativo del Gateway: el perfil
predeterminado usa `openclaw-YYYY-MM-DD.log`, mientras que los perfiles con nombre usan
`openclaw-<profile>-YYYY-MM-DD.log` (por ejemplo,
`openclaw-dev-YYYY-MM-DD.log`).

## Comportamiento de reserva y recuperación

- Si el Gateway de bucle local implícito solicita emparejamiento, cierra la conexión durante el establecimiento o agota el tiempo de espera antes de que `logs.tail` responda, `openclaw logs` recurre automáticamente al archivo de registro configurado del Gateway. Los destinos `--url` explícitos nunca usan este mecanismo de reserva.
- `--follow` no recurre a ese archivo configurado tras un fallo de RPC del Gateway local implícito, ya que un archivo paralelo obsoleto podría generar confusión durante un seguimiento en vivo. En Linux, usa en su lugar el diario del Gateway activo de systemd del usuario mediante el PID cuando está disponible (e imprime la fuente seleccionada); de lo contrario, continúa reintentando la conexión con el Gateway en vivo.
- Durante `--follow`, las desconexiones transitorias (cierre de WebSocket, tiempo de espera agotado o pérdida de conexión) activan la reconexión automática con espera exponencial: hasta 8 reintentos, con un máximo de 30s entre intentos. En cada reintento se imprime una advertencia en stderr y, cuando un sondeo se completa correctamente, se imprime una vez un aviso `[logs] gateway reconnected`. En el modo `--json`, ambos se emiten como registros `{"type":"notice"}` en stderr. Los errores no recuperables (fallo de autenticación o configuración incorrecta) siguen provocando una salida inmediata.
- En el modo `--follow --json`, las transiciones de la fuente de registros se emiten como registros `{"type":"meta"}`. Realice el seguimiento de los cursores por `sourceKind`: un flujo puede pasar de la salida del archivo del Gateway (`sourceKind: "file"`) al mecanismo de reserva del diario local (`sourceKind: "journal"`, `localFallback: true`, con `service.pid`/`service.unit`) y volver a la salida del archivo del Gateway tras la recuperación. No presuponga una única fuente ni un único cursor estables durante toda la sesión y admita líneas superpuestas cuando la recuperación vuelva a reproducir el cursor del archivo del Gateway.

## Contenido relacionado

- [Descripción general de los registros](/es/logging)
- [CLI del Gateway](/es/cli/gateway)
- [Referencia de la CLI](/es/cli)
- [Registros del Gateway](/es/gateway/logging)
