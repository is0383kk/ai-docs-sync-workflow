---
read_when:
    - Cambiar los modos de autenticación o exposición del panel de control
summary: Acceso y autenticación al panel del Gateway (interfaz de control)
title: Panel de control
x-i18n:
    generated_at: "2026-07-26T05:58:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca531ad2943dfdee1cd90a4efdc1fb69c4517780e2be52237fd558b8638e7cd0
    source_path: web/dashboard.md
    workflow: 16
---

El panel del Gateway es la interfaz de control para navegador que se sirve en `/` de forma predeterminada (se puede sustituir con `gateway.controlUi.basePath`).

Apertura rápida (Gateway local):

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (o [http://localhost:18789/](http://localhost:18789/))
- Con `gateway.tls.enabled: true`, use `https://127.0.0.1:18789/` y `wss://127.0.0.1:18789` para el endpoint WebSocket.

Referencias clave:

- [Interfaz de control](/es/web/control-ui) para consultar el uso y las capacidades de la interfaz.
- [Tailscale](/es/gateway/tailscale) para la automatización de Serve/Funnel.
- [Superficies web](/es/web) para consultar los modos de vinculación y las notas de seguridad.

La autenticación se aplica durante el protocolo de enlace de WebSocket mediante la ruta de autenticación del Gateway configurada:

- `connect.params.auth.token`
- `connect.params.auth.password`
- Encabezados de identidad de Tailscale Serve cuando `gateway.auth.allowTailscale: true`
- Encabezados de identidad de proxy de confianza cuando `gateway.auth.mode: "trusted-proxy"`

Consulte `gateway.auth` en [Configuración del Gateway](/es/gateway/configuration).

<Warning>
La interfaz de control es una **superficie de administración** (chat, configuración y aprobaciones de ejecución). No la exponga públicamente. La interfaz conserva los tokens de la URL del panel en sessionStorage para la pestaña actual del navegador y la URL del Gateway seleccionada, y los elimina de la URL después de cargarla. Se recomienda usar localhost, Tailscale Serve o un túnel SSH.
</Warning>

## Vía rápida (recomendada)

- Después de la incorporación, la CLI abre automáticamente el panel e imprime un enlace limpio (sin token).
- Vuelva a abrirlo en cualquier momento: `openclaw dashboard` (copia el enlace, abre un navegador si es posible e imprime una indicación sobre SSH si no hay interfaz gráfica).
- Si fallan tanto la entrega mediante el portapapeles como la apertura en el navegador, `openclaw dashboard` sigue imprimiendo la URL limpia e indica que se debe añadir el token (de `OPENCLAW_GATEWAY_TOKEN` o `gateway.auth.token`) como clave de fragmento de URL `token`; nunca imprime el valor del token en los registros.
- Si la interfaz solicita autenticación mediante secreto compartido, pegue el token o la contraseña configurados en los ajustes de la interfaz de control.

## Conceptos básicos de autenticación (local frente a remota)

- **Localhost**: abra `http://127.0.0.1:18789/`.
- **TLS del Gateway**: cuando `gateway.tls.enabled: true`, los enlaces del panel y de estado usan `https://`, y los enlaces WebSocket de la interfaz de control usan `wss://`.
- **Origen del token de secreto compartido**: `gateway.auth.token` (o `OPENCLAW_GATEWAY_TOKEN`). `openclaw dashboard` puede pasarlo mediante un fragmento de URL para la inicialización única; la interfaz de control lo conserva en sessionStorage para la pestaña actual y la URL del Gateway seleccionada, no en localStorage.
- **Token de ejecución por falta de configuración**: si al iniciarse se indica que se generó un token de ejecución, dicho token es efímero y no está disponible mediante `openclaw config get gateway.auth.token`. La interfaz de bucle invertido sigue requiriendo autenticación. Ejecute `openclaw doctor --generate-gateway-token`, reinicie el Gateway y, a continuación, pegue el token configurado en los ajustes de la interfaz de control.
- Si `gateway.auth.token` está gestionado mediante SecretRef, `openclaw dashboard` imprime, copia o abre intencionadamente una URL sin token para evitar exponer tokens gestionados externamente en los registros del shell, el historial del portapapeles o los argumentos de apertura del navegador. Si la referencia no se puede resolver en el shell actual, sigue imprimiendo la URL sin token junto con instrucciones prácticas para configurar la autenticación.
- **Contraseña de secreto compartido**: use el valor configurado de `gateway.auth.password` (o `OPENCLAW_GATEWAY_PASSWORD`). El panel no conserva las contraseñas entre recargas.
- **Modos con identidad**: Tailscale Serve satisface la autenticación de la interfaz de control/WebSocket mediante encabezados de identidad cuando `gateway.auth.allowTailscale: true`; un proxy inverso con reconocimiento de identidad y fuera de la interfaz de bucle invertido satisface `gateway.auth.mode: "trusted-proxy"`. Ninguno requiere pegar un secreto compartido para WebSocket.
- **Fuera de localhost**: use Tailscale Serve, una vinculación mediante secreto compartido fuera de la interfaz de bucle invertido, un proxy inverso con reconocimiento de identidad fuera de la interfaz de bucle invertido con `gateway.auth.mode: "trusted-proxy"`, o un túnel SSH. Las API HTTP siguen usando autenticación mediante secreto compartido, salvo que se ejecute intencionadamente `gateway.auth.mode: "none"` con entrada privada o autenticación HTTP mediante proxy de confianza. Consulte [Superficies web](/es/web).

## Abrir en Telegram

Los bots de Telegram pueden abrir el panel como una Mini App de Telegram con `/dashboard`.

Requisitos:

- `gateway.tailscale.mode: "serve"` o `"funnel"` para que Telegram obtenga una URL HTTPS de la Mini App.
- El remitente de Telegram debe ser el propietario del bot: un ID numérico de usuario de Telegram en `commands.ownerAllowFrom` o el valor efectivo de `channels.telegram.allowFrom` de la cuenta seleccionada.
- Ejecute `/dashboard` en un mensaje directo con el bot. Las invocaciones en grupos solo indican que se debe abrir el comando en un mensaje directo y no incluyen ningún botón.
- Instalaciones con Docker: los modos Serve/Funnel requieren que el Gateway se vincule a la interfaz de bucle invertido junto a `tailscaled`, algo que la red en puente con puertos publicados no puede satisfacer. Ejecute el contenedor del Gateway con `network_mode: host` y monte el socket `tailscaled` del host (`/var/run/tailscale`), así como la CLI `tailscale`, dentro del contenedor.

La Mini App realiza una entrega única del propietario y redirige a la interfaz de control con un token de inicialización de corta duración. No expone ningún token compartido del Gateway en la URL.

Fuera del alcance de la v1:

- El iframe web de Telegram no es compatible.
- Tailscale Serve/Funnel es la única ruta de URL publicada compatible.

<a id="if-you-see-unauthorized-1008"></a>

## Si aparece "unauthorized" / 1008

- Confirme que se puede acceder al Gateway: en local, `openclaw status`; en remoto, cree el túnel SSH `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host` y, a continuación, abra `http://127.0.0.1:18789/`.
- Para `AUTH_TOKEN_MISMATCH`, los clientes pueden realizar un reintento de confianza con un token de dispositivo almacenado en caché cuando el Gateway devuelve indicaciones de reintento; ese reintento reutiliza los ámbitos aprobados almacenados en caché del token (los llamadores explícitos de `deviceToken`/`scopes` conservan el conjunto de ámbitos solicitado). Si la autenticación sigue fallando después de ese reintento, resuelva manualmente la divergencia del token.
- Para `AUTH_SCOPE_MISMATCH`, el token del dispositivo se reconoció, pero no incluye los ámbitos solicitados; vuelva a emparejarlo o apruebe el nuevo conjunto de ámbitos en lugar de rotar el token compartido del Gateway.
- Fuera de esa ruta de reintento, la precedencia de autenticación de la conexión es: token compartido o contraseña explícitos, luego `deviceToken` explícito, después el token de dispositivo almacenado y, por último, el token de inicialización.
- En la ruta asíncrona de Tailscale Serve, los intentos fallidos correspondientes al mismo `{scope, ip}` se serializan antes de que el limitador de autenticaciones fallidas los registre, por lo que un segundo reintento incorrecto simultáneo ya puede mostrar `retry later`.
- Para consultar los pasos de reparación de divergencias de tokens, consulte la [Lista de comprobación para recuperar divergencias de tokens](/es/cli/devices#token-drift-recovery-checklist).
- Recupere o proporcione el secreto compartido desde el host del Gateway:
  - Token: `openclaw config get gateway.auth.token`
  - Contraseña: resuelva el valor configurado de `gateway.auth.password` o `OPENCLAW_GATEWAY_PASSWORD`
  - Token gestionado mediante SecretRef: resuelva el proveedor externo de secretos o exporte `OPENCLAW_GATEWAY_TOKEN` en este shell y vuelva a ejecutar `openclaw dashboard`
  - Token de ejecución generado porque no se configuró ningún secreto compartido: ejecute `openclaw doctor --generate-gateway-token`, reinicie el Gateway y, a continuación, use el token configurado
- En los ajustes del panel, pegue el token o la contraseña en el campo de autenticación y, a continuación, conéctese.
- El selector de idioma de la interfaz se encuentra en **Settings -> General -> Language**, no en Appearance.

## Contenido relacionado

- [Interfaz de control](/es/web/control-ui)
- [WebChat](/es/web/webchat)
