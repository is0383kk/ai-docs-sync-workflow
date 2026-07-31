---
read_when:
    - Integración de la aplicación para Mac con el ciclo de vida del Gateway
summary: Ciclo de vida del Gateway en macOS (launchd)
title: Ciclo de vida del Gateway en macOS
x-i18n:
    generated_at: "2026-07-26T05:19:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89a27334afcecb322feb2732cf6282b4c286ef27828a1b57157f9d4fc161aed6
    source_path: platforms/mac/child-process.md
    workflow: 16
---

La app de macOS administra el Gateway mediante **launchd** de forma predeterminada y no
inicia el Gateway como proceso secundario. Primero intenta conectarse a un
Gateway que ya esté en ejecución en el puerto configurado; si no se puede acceder a ninguno,
habilita el servicio de launchd mediante la CLI externa `openclaw` (sin
runtime integrado). Esto permite un inicio automático fiable al iniciar sesión y el reinicio tras fallos.

El modo de proceso secundario (Gateway iniciado directamente por la app) **no se utiliza**
actualmente. Si se necesita una integración más estrecha con la interfaz de usuario, se debe ejecutar el Gateway manualmente en una
terminal.

## Comportamiento predeterminado (launchd)

- La app instala un LaunchAgent por usuario con la etiqueta `ai.openclaw.gateway` (o
  `ai.openclaw.<profile>` al usar `--profile`/`OPENCLAW_PROFILE`).
- Cuando el modo local está habilitado, la app se asegura de que el LaunchAgent esté cargado e
  inicia el Gateway si es necesario.
- Los registros se escriben en la ruta del registro del Gateway de launchd (visible en la configuración de depuración).

Comandos habituales:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Sustituya la etiqueta por `ai.openclaw.<profile>` cuando se ejecute un perfil con nombre.

## Compilaciones de desarrollo sin firmar

`scripts/restart-mac.sh --no-sign` sirve para realizar compilaciones locales rápidas sin claves
de firma. Para evitar que launchd apunte a un binario de retransmisión sin firmar, escribe
`~/.openclaw/disable-launchagent`.

Las ejecuciones firmadas de `scripts/restart-mac.sh` eliminan esta anulación si el marcador está
presente. Para restablecerla manualmente:

```bash
rm ~/.openclaw/disable-launchagent
```

## Modo de solo conexión

Para impedir que la app de macOS instale o administre launchd, iníciela con
`--attach-only` (o `--no-launchd`). Esto establece
`~/.openclaw/disable-launchagent`, por lo que la app solo se conecta a un Gateway que ya esté
en ejecución. Se puede activar o desactivar el mismo comportamiento en la configuración de depuración.

## Modo remoto

El modo remoto nunca inicia un Gateway local. La app utiliza un túnel SSH hacia el
host remoto y se conecta a través de ese túnel.

## Por qué se prefiere launchd

- Inicio automático al iniciar sesión.
- Semántica integrada de reinicio/KeepAlive.
- Registros y supervisión predecibles.

Si alguna vez vuelve a ser necesario un verdadero modo de proceso secundario, deberá documentarse como
un modo independiente, explícito y exclusivo para desarrollo.

## Contenido relacionado

- [App de macOS](/es/platforms/macos)
- [Guía operativa del Gateway](/es/gateway)
