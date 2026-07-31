---
read_when:
    - Está usando mensajes directos en modo de emparejamiento y necesita aprobar a los remitentes
summary: Referencia de la CLI para `openclaw pairing` (aprobar/listar solicitudes de vinculación)
title: Emparejamiento
x-i18n:
    generated_at: "2026-07-26T05:06:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e4c6c53f1a3eefe50b4b7a45fa535e9a05faabb50df1ba5195a7635ee13d9da0
    source_path: cli/pairing.md
    workflow: 16
---

# `openclaw pairing`

Apruebe o inspeccione las solicitudes de vinculación por mensaje directo para los canales que admiten vinculación (solo mensajes directos de chat; la vinculación de nodos o dispositivos usa `openclaw devices`).

Relacionado: [Flujo de vinculación](/es/channels/pairing)

Las mismas solicitudes pendientes se pueden revisar en la interfaz de control, en **Settings →
Channels → DM access requests**. La interfaz de control permite aprobarlas, notificar opcionalmente
al solicitante y descartarlas. Descartar elimina la solicitud actual, pero no
bloquea permanentemente al remitente.

## Comandos

```bash
openclaw pairing list telegram
openclaw pairing list --channel telegram --account work
openclaw pairing list telegram --json

openclaw pairing approve <code>
openclaw pairing approve telegram <code>
openclaw pairing approve --channel telegram --account work <code> --notify
```

## `pairing list`

Muestra las solicitudes de vinculación pendientes de un canal.

| Opción                  | Descripción                           |
| ----------------------- | ------------------------------------- |
| `[channel]`             | identificador de canal posicional                 |
| `--channel <channel>`   | identificador de canal explícito                   |
| `--account <accountId>` | identificador de cuenta para canales con varias cuentas |
| `--json`                | salida legible por máquina               |

Si hay configurados varios canales que admiten vinculación, indique un canal como argumento posicional o mediante `--channel`. Los canales de extensión funcionan siempre que el identificador del canal sea válido.

## `pairing approve`

Aprueba un código de vinculación pendiente y autoriza a ese remitente.

Uso:

- `openclaw pairing approve <channel> <code>`
- `openclaw pairing approve --channel <channel> <code>`
- `openclaw pairing approve <code>` cuando haya configurado exactamente un canal que admita vinculación

Opciones: `--channel <channel>`, `--account <accountId>`, `--notify` (envía una confirmación al solicitante a través del mismo canal).

### Configuración inicial del propietario

Si `commands.ownerAllowFrom` está vacío al aprobar un código de vinculación, la CLI también registra al remitente aprobado como propietario de comandos mediante una entrada específica del canal, como `telegram:123456789`. Esto solo configura al primer propietario; las aprobaciones de vinculación posteriores nunca sustituyen ni amplían `commands.ownerAllowFrom`. La interfaz de control presenta esta elevación como una casilla separada protegida mediante `operator.admin`, en lugar de aplicarla automáticamente.

El propietario de comandos es la cuenta del operador humano autorizada para ejecutar comandos exclusivos del propietario y aprobar acciones peligrosas como `/diagnostics`, `/export-session`, `/export-trajectory`, `/config` y las aprobaciones de ejecución. La vinculación solo permite que un remitente se comunique con el agente; por sí sola, no concede privilegios de propietario más allá de esta configuración inicial única.

Si aprobó a un remitente antes de que existiera esta configuración inicial, ejecute `openclaw doctor`; este comando advierte cuando no hay ningún propietario de comandos configurado y muestra el comando `openclaw config set commands.ownerAllowFrom ...` exacto para corregirlo.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Vinculación de canales](/es/channels/pairing)
