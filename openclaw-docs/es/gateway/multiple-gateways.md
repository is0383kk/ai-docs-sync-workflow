---
read_when:
    - Ejecutar más de un Gateway en la misma máquina
    - Se necesitan configuraciones, estados y puertos aislados para cada Gateway
summary: Ejecutar varios Gateways de OpenClaw en un mismo host (aislamiento, puertos y perfiles)
title: Varios gateways
x-i18n:
    generated_at: "2026-07-26T05:07:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 655fa865a98064d7c017a7c2eb08ea9a9683002d96a3dbe45a8c16cbd3c86ba1
    source_path: gateway/multiple-gateways.md
    workflow: 16
---

La mayoría de las configuraciones necesitan un Gateway: un solo Gateway gestiona varias conexiones de mensajería y agentes. Ejecute Gateways independientes con perfiles/puertos aislados solo cuando necesite un aislamiento más estricto o redundancia (por ejemplo, un bot de rescate).

## Inicio rápido del bot de rescate

La configuración más sencilla para un bot de rescate:

- Mantenga el bot principal en el perfil predeterminado.
- Ejecute el bot de rescate en `--profile rescue`, con su propio token de bot de Telegram.
- Asigne al bot de rescate un puerto base diferente, por ejemplo, `19789`.

Esto permite que el bot de rescate depure o aplique cambios de configuración si el bot principal deja de funcionar. Deje al menos 20 puertos entre los puertos base para que los puertos derivados del navegador/CDP nunca entren en conflicto.

```bash
# Bot de rescate (bot de Telegram independiente, perfil independiente, puerto 19789)
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

Si el bot principal ya está en ejecución, normalmente esto es todo lo necesario. Si la incorporación ya instaló el servicio de rescate, omita el `gateway install` final.

Durante `openclaw --profile rescue onboard`:

- Use un token de bot de Telegram independiente, dedicado a la cuenta de rescate (fácil de mantener solo para operadores, independiente de la instalación del canal/aplicación del bot principal y una ruta de recuperación sencilla basada en mensajes directos).
- Mantenga el nombre de perfil `rescue`.
- Use un puerto base que sea al menos 20 mayor que el del bot principal.
- Acepte el espacio de trabajo de rescate predeterminado, a menos que ya gestione uno por su cuenta.

### Qué cambia `--profile rescue onboard`

`--profile rescue onboard` ejecuta el flujo normal de incorporación, pero escribe todo en un perfil independiente, por lo que el bot de rescate obtiene sus propios elementos:

- Archivo de perfil/configuración
- Directorio de estado
- Espacio de trabajo (predeterminado: `~/.openclaw/workspace-rescue`)
- Nombre del servicio administrado
- Puerto base (más los puertos derivados)
- Token de bot de Telegram

Por lo demás, las indicaciones son idénticas a las de la incorporación normal.

## Configuración general de varios Gateways

El mismo patrón de aislamiento funciona para cualquier par o grupo de Gateways en un host: asigne a cada Gateway adicional su propio perfil con nombre y puerto base:

```bash
# principal (perfil predeterminado)
openclaw setup
openclaw gateway --port 18789

# gateway adicional
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

También se pueden usar perfiles con nombre en ambos lados:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

Los servicios siguen el mismo patrón:

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

Use el inicio rápido del bot de rescate para disponer de una vía alternativa para operadores; use el patrón general de perfiles para varios Gateways de larga duración en distintos canales, inquilinos, espacios de trabajo o roles operativos.

## Lista de comprobación del aislamiento

Mantenga estos valores únicos para cada instancia de Gateway:

| Configuración                      | Propósito                              |
| ---------------------------- | ------------------------------------ |
| `OPENCLAW_CONFIG_PATH`       | Archivo de configuración por instancia             |
| `OPENCLAW_STATE_DIR`         | Sesiones, credenciales y cachés por instancia |
| `agents.defaults.workspace`  | Raíz del espacio de trabajo por instancia          |
| `gateway.port` (o `--port`) | Único por instancia                  |
| Puertos derivados del navegador/CDP    | Consulte a continuación                            |

Compartir cualquiera de estos elementos provoca conflictos de configuración, estado o puertos. El inicio del Gateway
exige que cada directorio de estado tenga un propietario único, incluso cuando
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` omite la instancia única por configuración.

## Asignación de puertos (derivados)

Puerto base = `gateway.port` (o `OPENCLAW_GATEWAY_PORT` / `--port`).

- Puerto del servicio de control del navegador = base + 2 (solo bucle local).
- El host de Canvas se sirve en el propio servidor HTTP del Gateway (el mismo puerto que `gateway.port`).
- Los puertos CDP del perfil del navegador se asignan automáticamente desde `browser control port + 9` hasta `+ 108`.

Si sobrescribe cualquiera de estos valores en la configuración o el entorno, debe mantenerlos únicos para cada instancia.

## Notas sobre el navegador/CDP (error frecuente)

- **No** fije `browser.cdpUrl` en el mismo valor para varias instancias.
- Cada instancia necesita su propio puerto de control del navegador y rango de CDP (derivados de su puerto del Gateway).
- Para establecer puertos CDP explícitos, configure `browser.profiles.<name>.cdpPort` para cada instancia.
- Para Chrome remoto, use `browser.profiles.<name>.cdpUrl` (por perfil y por instancia).

## Ejemplo manual con variables de entorno

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## Comprobaciones rápidas

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

- `gateway status --deep` detecta servicios launchd/systemd/schtasks obsoletos de instalaciones anteriores.
- El texto de advertencia de `gateway probe`, como `multiple reachable gateway identities detected`, solo es esperado cuando se ejecuta intencionadamente más de un Gateway aislado o cuando OpenClaw no puede demostrar que los destinos de sondeo accesibles son el mismo Gateway. Un túnel SSH, una URL de proxy o una URL remota configurada que apuntan al mismo Gateway constituyen un solo Gateway con varios transportes, aunque los puertos de transporte sean diferentes.

## Contenido relacionado

- [Guía operativa del Gateway](/es/gateway)
- [Bloqueo del Gateway](/es/gateway/gateway-lock)
- [Configuración](/es/gateway/configuration)
