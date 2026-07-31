---
read_when:
    - Quieres acceder al Gateway mediante Tailscale
    - Quieres la interfaz de control del navegador y la edición de la configuración
summary: 'Superficies web del Gateway: interfaz de control, modos de enlace y seguridad'
title: Web
x-i18n:
    generated_at: "2026-07-26T04:58:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 413fb029d95241f5c6043b28825727cdee52b2fa8cbe998fbbd6e3ff7b81467b
    source_path: web/index.md
    workflow: 16
---

El Gateway sirve una pequeña **IU de control en el navegador** (Vite + Lit) desde el mismo puerto que el WebSocket del Gateway:

- predeterminado: `http://<host>:18789/`
- con `gateway.tls.enabled: true`: `https://<host>:18789/`
- prefijo opcional: establezca `gateway.controlUi.basePath` (p. ej., `/openclaw`)

Las capacidades se describen en [IU de control](/es/web/control-ui). Esta página abarca los modos de enlace, la seguridad y otras superficies accesibles desde la web.

## Configuración (activada de forma predeterminada)

La IU de control está **activada de forma predeterminada** cuando los recursos están presentes (`dist/control-ui`):

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // basePath opcional
  },
}
```

## Webhooks

Cuando `hooks.enabled=true`, el Gateway también expone un endpoint de webhook en el mismo servidor HTTP. Consulte `hooks` en la [referencia de configuración del Gateway](/es/gateway/configuration-reference#hooks) para obtener información sobre la autenticación y las cargas útiles.

## RPC HTTP de administración

`POST /api/v1/admin/rpc` expone determinados métodos del plano de control del Gateway mediante HTTP. Está desactivado de forma predeterminada y solo se registra cuando el plugin `admin-http-rpc` está activado. Consulte [RPC HTTP de administración](/es/plugins/admin-http-rpc) para conocer el modelo de autenticación, los métodos permitidos y la comparación con la API WebSocket.

## Acceso mediante Tailscale

<Tabs>
  <Tab title="Serve integrado (recomendado)">
    Mantenga el Gateway en la interfaz de bucle invertido y permita que Tailscale Serve actúe como proxy:

    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "serve" },
      },
    }
    ```

    Inicie el Gateway:

    ```bash
    openclaw gateway
    ```

    Abra `https://<magicdns>/` (o el `gateway.controlUi.basePath` configurado).

  </Tab>
  <Tab title="Enlace a la tailnet + token">
    ```json5
    {
      gateway: {
        bind: "tailnet",
        controlUi: { enabled: true },
        auth: { mode: "token", token: "your-token" },
      },
    }
    ```

    Inicie el Gateway (este ejemplo sin bucle invertido utiliza autenticación mediante token de secreto compartido):

    ```bash
    openclaw gateway
    ```

    Abra `http://<tailscale-ip>:18789/` (o el `gateway.controlUi.basePath` configurado).

  </Tab>
  <Tab title="Internet pública (Funnel)">
    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "funnel" },
        auth: { mode: "password" }, // o OPENCLAW_GATEWAY_PASSWORD
      },
    }
    ```

    `tailscale.mode: "funnel"` requiere `gateway.auth.mode: "password"`; tanto Serve como Funnel requieren `gateway.bind: "loopback"`.

  </Tab>
</Tabs>

## Notas de seguridad

- La autenticación del Gateway es obligatoria de forma predeterminada: token, contraseña, proxy de confianza o encabezados de identidad de Tailscale Serve cuando estén activados.
- Los enlaces que no sean de bucle invertido siguen **requiriendo** autenticación del Gateway: autenticación mediante token/contraseña o un proxy inverso con reconocimiento de identidad y `gateway.auth.mode: "trusted-proxy"`.
- El asistente de incorporación crea autenticación mediante secreto compartido de forma predeterminada y normalmente genera un token del Gateway, incluso en la interfaz de bucle invertido.
- En el modo de secreto compartido, la IU envía `connect.params.auth.token` o `connect.params.auth.password` durante el protocolo de enlace de WebSocket.
- Con `gateway.tls.enabled: true`, los auxiliares locales del panel de control y de estado generan direcciones URL `https://` y direcciones URL de WebSocket `wss://`.
- En los modos que incorporan identidad (Tailscale Serve, `trusted-proxy`), la comprobación de autenticación de WebSocket se satisface mediante los encabezados de la solicitud en lugar de un secreto compartido.
- Para implementaciones públicas de la IU de control que no usen la interfaz de bucle invertido, establezca `gateway.controlUi.allowedOrigins` explícitamente (orígenes completos). Las cargas privadas del mismo origen se aceptan sin este valor para la interfaz de bucle invertido, RFC1918/enlace local, `.local`, `.ts.net` y hosts CGNAT de Tailscale.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback: true` activa el uso alternativo del origen basado en el encabezado Host; esto supone una reducción peligrosa de la seguridad.
- Con Serve, los encabezados de identidad de Tailscale satisfacen la autenticación de la IU de control/WebSocket cuando `gateway.auth.allowTailscale: true` (no se requiere token ni contraseña). Los endpoints de la API HTTP no utilizan los encabezados de identidad de Tailscale; siempre siguen el modo de autenticación HTTP normal del Gateway. Establezca `gateway.auth.allowTailscale: false` para exigir credenciales explícitas incluso mediante Serve. Este flujo sin token presupone que el propio host del Gateway es de confianza. Consulte [Tailscale](/es/gateway/tailscale) y [Seguridad](/es/gateway/security).

## Compilación de la IU

El Gateway sirve archivos estáticos desde `dist/control-ui`:

```bash
pnpm ui:build
```
