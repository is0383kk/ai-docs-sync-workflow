---
read_when:
    - Ejecutar OpenClaw Gateway en WSL2 mientras Chrome se ejecuta en Windows
    - Observación de errores superpuestos del navegador y de la interfaz de control en WSL2 y Windows
    - Elección entre Chrome MCP local al host y CDP remoto sin procesar en configuraciones con hosts separados
summary: Solucionar problemas de Gateway en WSL2 y CDP remoto de Chrome en Windows por capas
title: Solución de problemas de WSL2 + Windows + CDP remoto de Chrome
x-i18n:
    generated_at: "2026-07-26T04:53:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 66ec4ed5bfccc66b594a43d56296c69242e8b9cf50b36c6cb3990b1d6ea58faa
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 16
---

En la configuración habitual con hosts separados, el Gateway de OpenClaw se ejecuta dentro de WSL2, Chrome se ejecuta
en Windows y el control del navegador debe atravesar el límite entre WSL2 y Windows. Pueden surgir varios
problemas independientes a la vez (consulte el
[issue #39369](https://github.com/openclaw/openclaw/issues/39369)): el transporte
CDP, la seguridad del origen de la interfaz de control y el token/emparejamiento pueden fallar cada uno por
separado y producir errores de apariencia similar. Recorra en orden las capas
siguientes en lugar de intentar adivinar cuál está fallando.

## Elija primero el modo de navegador adecuado

### Opción 1: CDP remoto directo de WSL2 a Windows

Utilice un perfil de navegador remoto que apunte desde WSL2 a un endpoint CDP
de Chrome en Windows. Elija esta opción cuando el Gateway permanezca dentro de WSL2, Chrome se ejecute en
Windows y el control del navegador deba atravesar el límite entre WSL2 y Windows.

### Opción 2: MCP de Chrome local al host

Utilice el controlador `existing-session` (perfil `user`) únicamente cuando el Gateway se ejecute
en el mismo host que Chrome, se quiera usar el estado local del navegador con la sesión iniciada, no se
necesite transporte del navegador entre hosts y no se necesiten `responsebody`,
exportación a PDF, interceptación de descargas ni acciones por lotes (los perfiles de MCP de Chrome no
las admiten).

Para Gateway en WSL2 + Chrome en Windows, utilice CDP remoto directo. MCP de Chrome es
local al host, no un puente de WSL2 a Windows.

## Arquitectura funcional

- WSL2 ejecuta el Gateway en `127.0.0.1:18789`
- Windows abre la interfaz de control en un navegador normal en `http://127.0.0.1:18789/`
- Chrome en Windows expone un endpoint CDP en el puerto `9222`
- WSL2 puede acceder a ese endpoint CDP de Windows
- OpenClaw dirige un perfil de navegador a la dirección accesible desde WSL2

## Regla crítica para la interfaz de control

Cuando la interfaz se abra desde Windows, utilice el localhost de Windows salvo que haya una
configuración HTTPS deliberada:

```text
http://127.0.0.1:18789/
```

No utilice de forma predeterminada una IP de LAN. HTTP sin cifrar en una dirección de LAN o tailnet puede
activar comportamientos de origen no seguro o autenticación de dispositivo ajenos al propio CDP. Consulte
[Interfaz de control](/es/web/control-ui).

## Validación por capas

Proceda de arriba abajo; no se salte pasos. Corregir una capa puede dejar todavía
visible un error diferente de una capa posterior.

### Capa 1: verifique que Chrome proporciona CDP en Windows

```powershell
chrome.exe --remote-debugging-port=9222 --user-data-dir="$env:LOCALAPPDATA\OpenClaw\ChromeCDP"
```

Chrome 136 y versiones posteriores ignoran los modificadores de línea de comandos de depuración remota para el
directorio de datos predeterminado de Chrome. Utilice un directorio de datos separado y no predeterminado, como
se muestra arriba. Consulte el
[cambio de seguridad de la depuración remota](https://developer.chrome.com/blog/remote-debugging-port)
de Chrome. Esto no permite controlar de forma remota el perfil normal de Chrome con la sesión iniciada.

Desde Windows, verifique primero el propio Chrome:

```powershell
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://127.0.0.1:9222/json/list
```

Si esto falla, diagnostique los listeners de Windows indicados a continuación. OpenClaw aún no es el
problema.

#### Diagnostique IPv4 e IPv6 antes de cambiar portproxy

Chromium intenta vincular primero la depuración remota a `127.0.0.1` y recurre a
`[::1]` únicamente si falla la vinculación IPv4. Una regla persistente de `v4tov4` que escuche en
`127.0.0.1:9222` puede ocupar ese endpoint antes de que se inicie Chrome. Chrome pasa entonces
a `[::1]:9222`, mientras que la regla antigua reenvía el tráfico IPv4 a
su propio listener y devuelve una respuesta vacía.

Compruebe desde Windows los listeners y las reglas de proxy reales en lugar de deducirlos
a partir de la versión de Chrome:

```powershell
netstat -ano | findstr :9222
netsh interface portproxy show all
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://[::1]:9222/json/version
```

Utilice `tasklist /fi "PID eq <PID>"` para cada PID de `netstat`.

- Si `chrome.exe` responde en `127.0.0.1`, elimine cualquier regla portproxy que también
  escuche en `127.0.0.1:9222`. Reenvíe únicamente la dirección del adaptador de Windows accesible desde WSL2
  a `127.0.0.1`.
- Si `chrome.exe` responde únicamente en `[::1]`, dirija el listener accesible desde WSL2 a
  `::1` con `v4tov6` en lugar de reenviar a una dirección IPv4 sin utilizar:

  ```powershell
  netsh interface portproxy add v4tov6 listenaddress=WINDOWS_HOST_OR_IP listenport=9222 connectaddress=::1 connectport=9222
  ```

Vincule el listener a la dirección del adaptador que necesita WSL2. No exponga el puerto
CDP en `0.0.0.0`, una dirección de LAN ni una dirección de tailnet: CDP concede el control de
la sesión del navegador.

### Capa 2: verifique que WSL2 puede acceder a ese endpoint de Windows

Desde WSL2, pruebe la dirección exacta que se vaya a utilizar en `cdpUrl`:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

Resultado correcto:

- `/json/version` devuelve JSON con metadatos de Browser / Protocol-Version
- `/json/list` devuelve JSON (una matriz vacía es válida si no hay páginas abiertas)

Si esto falla, Windows aún no expone el puerto a WSL2, la dirección es
incorrecta para el lado de WSL2 o falta el firewall, el reenvío de puertos o el proxy. Corrija
eso antes de modificar la configuración de OpenClaw.

### Capa 3: configure el perfil de navegador correcto

Dirija OpenClaw a la dirección accesible desde WSL2:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

Notas:

- utilice la dirección accesible desde WSL2, no una que solo funcione en Windows
- mantenga `attachOnly: true` para navegadores administrados externamente
- `cdpUrl` puede ser `http://`, `https://`, `ws://` o `wss://`
- utilice HTTP(S) cuando quiera que OpenClaw detecte `/json/version`
- utilice WS(S) únicamente cuando el proveedor del navegador proporcione una URL directa del socket
  de DevTools
- pruebe la misma URL con `curl` antes de esperar que OpenClaw funcione correctamente

### Capa 4: verifique por separado la capa de la interfaz de control

Abra `http://127.0.0.1:18789/` desde Windows y, a continuación, verifique:

- el origen de la página coincide con lo que espera `gateway.controlUi.allowedOrigins`
- la autenticación mediante token o el emparejamiento están configurados correctamente
- no se está diagnosticando un problema de autenticación de la interfaz de control como si fuera un problema del navegador

Página útil: [Interfaz de control](/es/web/control-ui).

### Capa 5: verifique el control integral del navegador

Desde WSL2:

```bash
openclaw browser --browser-profile remote open https://example.com
openclaw browser --browser-profile remote tabs
```

Resultado correcto:

- la pestaña se abre en Chrome en Windows
- `browser tabs` devuelve el objetivo
- las acciones posteriores (`snapshot`, `screenshot`, `navigate`) funcionan desde el mismo
  perfil

## Errores comunes que pueden inducir a error

| Mensaje                                                                                 | Significado                                                                                                                                                                           |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `control-ui-insecure-auth`                                                              | problema del origen/contexto seguro de la interfaz, no del transporte CDP                                                                                                                     |
| `token_missing`                                                                         | problema de configuración de autenticación                                                                                                                                                        |
| `pairing required`                                                                      | problema de aprobación del dispositivo                                                                                                                                                           |
| `Remote CDP for profile "remote" is not reachable`                                      | WSL2 no puede acceder al `cdpUrl` configurado                                                                                                                                         |
| respuesta CDP vacía / `other side closed` mediante un portproxy                               | discrepancia en el listener de Windows o bucle sobre sí mismo; inspeccione ambas familias de loopback y `netsh interface portproxy show all`                                                                 |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable` | el endpoint HTTP respondió, pero no se pudo abrir el WebSocket de DevTools                                                                                                        |
| valores obsoletos de área de visualización / modo oscuro / configuración regional / modo sin conexión después de una sesión remota          | ejecute `openclaw browser --browser-profile remote stop` para cerrar la sesión y liberar la conexión de Playwright/CDP almacenada en caché sin reiniciar el Gateway ni el navegador externo |
| tiempo de espera agotado durante la comprobación de acceso a CDP                                                         | normalmente sigue siendo un problema de acceso a CDP o un endpoint remoto lento o inaccesible                                                                                                             |
| `Playwright page enumeration timed out after 3000ms`                                    | se estableció la conexión con el CDP remoto, pero se bloqueó la lectura persistente de su pestaña                                                                                                                     |
| `No Chrome tabs found for profile="user"`                                               | se seleccionó un perfil MCP de Chrome local sin pestañas disponibles en el host local                                                                                                          |

## Lista de comprobación para un diagnóstico rápido

1. Windows: ¿cuál de `127.0.0.1` o `[::1]` responde en `/json/version` y
   pertenece ese listener a `chrome.exe`?
2. WSL2: ¿funciona `curl http://WINDOWS_HOST_OR_IP:9222/json/version`?
3. Configuración de OpenClaw: ¿utiliza `browser.profiles.<name>.cdpUrl` esa dirección exacta
   accesible desde WSL2?
4. Interfaz de control: ¿se está abriendo `http://127.0.0.1:18789/` en lugar de una IP de LAN?
5. ¿Se está intentando utilizar `existing-session` entre WSL2 y Windows en lugar
   de CDP remoto directo?

Verifique primero localmente el endpoint de Chrome en Windows, luego verifique el mismo endpoint
desde WSL2 y solo entonces diagnostique la configuración de OpenClaw o la autenticación de la interfaz de control.

## Temas relacionados

- [Navegador](/es/tools/browser)
- [Inicio de sesión en el navegador](/es/tools/browser-login)
- [Solución de problemas del navegador en Linux](/es/tools/browser-linux-troubleshooting)
