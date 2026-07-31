---
read_when:
    - Todavía se usa `openclaw daemon ...` en scripts
    - Necesita comandos para gestionar el ciclo de vida del servicio (install/start/stop/restart/status)
summary: Referencia de la CLI para `openclaw daemon` (alias heredado para la gestión del servicio Gateway)
title: Demonio
x-i18n:
    generated_at: "2026-07-26T04:32:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

Alias heredado para la gestión del servicio Gateway. `openclaw daemon ...` se corresponde con los mismos comandos de control del servicio que `openclaw gateway ...`. Se recomienda [`openclaw gateway`](/es/cli/gateway) para consultar la documentación y los ejemplos actuales.

## Uso

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## Subcomandos y opciones

| Subcomando  | Opciones                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable` (solo launchd: suprime de forma persistente KeepAlive/RunAtLoad hasta el siguiente inicio) |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`: muestra el estado de instalación del servicio (launchd/systemd/schtasks) y comprueba el estado del Gateway.
- `install`: instala el servicio; `--force` vuelve a instalar o sobrescribe una instalación existente.
- `restart --safe`: solicita al Gateway en ejecución que compruebe previamente el trabajo activo y programe un único reinicio consolidado después de que finalice el trabajo, con un límite de 5 minutos. Cuando se agota ese plazo, el reinicio se fuerza de todos modos. `restart` sin opciones utiliza directamente el gestor de servicios; `--force` fuerza la ejecución inmediata.
- `restart --safe --skip-deferral`: omite el mecanismo de aplazamiento por trabajo activo para que el Gateway se reinicie inmediatamente, incluso cuando se notifican bloqueos. Requiere `--safe`.

## Notas

- `status` resuelve los SecretRefs de autenticación configurados para autenticar la comprobación cuando es posible. Si un SecretRef obligatorio no se resuelve, `status --json` informa de `rpc.authWarning`; proporcione `--token`/`--password` explícitamente o resuelva primero el origen del secreto. Las advertencias de autenticación no resuelta se suprimen una vez que la comprobación tiene éxito por lo demás.
- `status --deep` añade un análisis de mejor esfuerzo a nivel del sistema para buscar otros servicios similares a Gateway (muestra indicaciones de limpieza; se sigue recomendando un Gateway por máquina) y ejecuta la validación de la configuración en modo compatible con plugins, mostrando advertencias de los manifiestos de plugins que la ruta rápida predeterminada omite.
- En instalaciones de systemd en Linux, las comprobaciones de discrepancias de tokens inspeccionan los orígenes de las unidades `Environment=` y `EnvironmentFile=`.
- Las comprobaciones de discrepancias de tokens resuelven los SecretRefs de `gateway.auth.token` mediante el entorno combinado de ejecución (primero el entorno del comando del servicio y, después, el entorno del proceso). Si la autenticación mediante token no está activa de forma efectiva (`gateway.auth.mode` con el valor `password`/`none`/`trusted-proxy`, o sin definir cuando la contraseña puede prevalecer), se omite la resolución del token de configuración.
- `install` valida que un `gateway.auth.token` gestionado mediante SecretRef pueda resolverse, pero nunca conserva el valor resuelto en los metadatos del entorno del servicio; si no puede resolverlo, la instalación se interrumpe de forma segura.
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está definido, `install` bloquea la operación hasta que se establezca explícitamente el modo.
- En macOS, `install` mantiene los plists de LaunchAgent y el archivo de entorno o contenedor generado accesibles únicamente para el propietario (modo `0600`/`0700`), en lugar de insertar secretos en `EnvironmentVariables`.
- Para ejecutar varios Gateways en un mismo host, se deben aislar los puertos, la configuración y el estado, y los espacios de trabajo. Consulte [Varios gateways](/es/gateway#multiple-gateways-same-host).

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Guía operativa del Gateway](/es/gateway)
