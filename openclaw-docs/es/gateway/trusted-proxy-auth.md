---
read_when:
    - Ejecución de OpenClaw detrás de un proxy con reconocimiento de identidad
    - Configuración de Pomerium, Caddy o nginx con OAuth delante de OpenClaw
    - Corrección de errores de WebSocket 1008 por falta de autorización en configuraciones con proxy inverso
    - Decidir dónde configurar HSTS y otras cabeceras de refuerzo de seguridad HTTP
sidebarTitle: Trusted proxy auth
summary: Delegar la autenticación del Gateway a un proxy inverso de confianza (Pomerium, Caddy, nginx + OAuth)
title: Autenticación mediante proxy de confianza
x-i18n:
    generated_at: "2026-07-26T04:39:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**Función sensible para la seguridad.** Este modo delega toda la autenticación en el proxy inverso. Una configuración incorrecta puede exponer el Gateway a accesos no autorizados. Lea atentamente esta página antes de habilitarlo.
</Warning>

## Cuándo usarlo

- OpenClaw se ejecuta detrás de un **proxy con reconocimiento de identidad** (Pomerium, Caddy + OAuth, nginx + oauth2-proxy, Traefik + autenticación reenviada).
- El proxy gestiona toda la autenticación y transmite la identidad del usuario mediante encabezados.
- Se utiliza un entorno de Kubernetes o contenedores en el que el proxy es la única ruta al Gateway.
- Se producen errores de WebSocket `1008 unauthorized` porque los navegadores no pueden transmitir tokens en las cargas útiles de WS.

## Cuándo NO usarlo

- El proxy no autentica a los usuarios (solo es un terminador TLS o un balanceador de carga).
- Existe alguna ruta al Gateway que evita el proxy (aperturas en el cortafuegos, acceso desde la red interna).
- No se sabe con certeza si el proxy elimina o sobrescribe correctamente los encabezados reenviados.
- Solo se necesita acceso personal para un único usuario (considere en su lugar Tailscale Serve + bucle invertido).

## Cómo funciona

<Steps>
  <Step title="El proxy autentica al usuario">
    El proxy inverso autentica a los usuarios (OAuth, OIDC, SAML, etc.).
  </Step>
  <Step title="El proxy añade un encabezado de identidad">
    El proxy añade un encabezado con la identidad del usuario autenticado (p. ej., `x-forwarded-user: nick@example.com`).
  </Step>
  <Step title="El Gateway verifica el origen de confianza">
    OpenClaw comprueba que la solicitud proceda de una **IP de proxy de confianza** (`gateway.trustedProxies`) y que no sea la dirección de bucle invertido ni una dirección de interfaz local del propio Gateway.
  </Step>
  <Step title="El Gateway extrae la identidad">
    OpenClaw lee los encabezados obligatorios y, a continuación, la identidad del usuario del encabezado configurado.
  </Step>
  <Step title="Autorizar">
    Si todas las comprobaciones son correctas y el usuario supera `allowUsers` (cuando se ha configurado), se autoriza la solicitud.
  </Step>
</Steps>

## Configuración

```json5
{
  gateway: {
    // De forma predeterminada, la autenticación mediante proxy de confianza espera que la IP de origen del proxy no sea de bucle invertido
    bind: "lan",

    // CRÍTICO: Añada aquí únicamente las IP del proxy
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // Encabezado que contiene la identidad del usuario autenticado (obligatorio)
        userHeader: "x-forwarded-user",

        // Opcional: encabezados que DEBEN estar presentes (verificación del proxy)
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // Opcional: restringir a usuarios específicos (vacío = permitir a todos)
        allowUsers: ["nick@example.com", "admin@company.org"],

        // Opcional: permitir un proxy de bucle invertido en el mismo host tras habilitarlo explícitamente
        allowLoopback: false,

        // Opcional: permitir que los usuarios autenticados por el proxy registren nuevos dispositivos de navegador
        deviceAutoApprove: {
          enabled: false,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

<Warning>
**Reglas de ejecución, por orden de evaluación**

1. La IP de origen de la solicitud debe coincidir con `gateway.trustedProxies` (con compatibilidad con CIDR); de lo contrario, se rechaza (`trusted_proxy_untrusted_source`).
2. Las solicitudes cuyo origen sea de bucle invertido (`127.0.0.1`, `::1`) se rechazan a menos que `gateway.auth.trustedProxy.allowLoopback = true` y la dirección de bucle invertido también figure en `trustedProxies` (`trusted_proxy_loopback_source`). Esta comprobación se ejecuta antes que las comprobaciones de encabezados, por lo que un origen de bucle invertido falla de este modo aunque también falten encabezados obligatorios.
3. Los orígenes que no sean de bucle invertido y coincidan con una de las direcciones de interfaz de red local del host del Gateway se rechazan como protección contra la suplantación (`trusted_proxy_local_interface_source`). Si falla la propia detección de interfaces, también se rechaza la solicitud (`trusted_proxy_local_interface_check_failed`).
4. `requiredHeaders` y `userHeader` deben estar presentes y no estar en blanco.
5. `allowUsers`, si no está vacío, debe incluir al usuario extraído.

**La evidencia de encabezados reenviados prevalece sobre el carácter local del bucle invertido para la alternativa local directa.** Si una solicitud llega mediante bucle invertido, pero incluye un encabezado `Forwarded`, cualquier `X-Forwarded-*` o `X-Real-IP`, dicha evidencia impide que utilice la autenticación alternativa con contraseña local directa y el control mediante identidad de dispositivo, aunque la autenticación mediante proxy de confianza siga fallando por tratarse de un bucle invertido.

`allowLoopback` confía en los procesos locales del host del Gateway en la misma medida que en el proxy inverso. Habilítelo únicamente cuando el Gateway siga protegido mediante cortafuegos contra el acceso remoto directo y el proxy local elimine o sobrescriba los encabezados de identidad proporcionados por el cliente.

Los clientes internos del Gateway que no pasen por el proxy inverso deben utilizar `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`, no encabezados de identidad de proxy de confianza. Las implementaciones de la interfaz de control que no sean de bucle invertido siguen necesitando `gateway.controlUi.allowedOrigins` de forma explícita.
</Warning>

### Referencia de configuración

<ParamField path="gateway.trustedProxies" type="string[]" required>
  Matriz de direcciones IP de proxy (o CIDR) en las que se confía. Se rechazan las solicitudes procedentes de otras IP.
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  Debe ser `"trusted-proxy"`.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  Nombre del encabezado que contiene la identidad del usuario autenticado.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  Encabezados adicionales que deben estar presentes para que la solicitud se considere de confianza.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  Lista de usuarios permitidos por identidad. Si está vacía, se permiten todos los usuarios autenticados.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  Compatibilidad opcional con proxies inversos de bucle invertido en el mismo host.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  Aprueba automáticamente nuevas identidades de dispositivo de la interfaz de control y WebChat tras la autenticación mediante proxy de confianza.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  Ámbitos máximos concedidos a un dispositivo de navegador aprobado automáticamente. Incluir explícitamente `operator.admin` permite que cada usuario autenticado por el proxy solicite una concesión automática de dispositivo con privilegios administrativos completos, hace que las solicitudes sin ámbitos reciban automáticamente privilegios administrativos completos y activa el hallazgo de auditoría de seguridad CRÍTICO `gateway.trusted_proxy_device_auto_approve_admin`, además de una advertencia al iniciar el Gateway.
</ParamField>

<Warning>
Habilite `allowLoopback` únicamente cuando el proxy inverso local sea el límite de confianza previsto. Cualquier proceso local que pueda conectarse al Gateway puede intentar enviar encabezados de identidad de proxy; por tanto, mantenga privado para el host el acceso directo al Gateway y exija encabezados controlados por el proxy, como `x-forwarded-proto`, o un encabezado de aserción firmado si el proxy lo admite.
</Warning>

## Aprobación automática de dispositivos

La autenticación mediante proxy de confianza puede utilizar opcionalmente la identidad del proxy como límite de aprobación para nuevos dispositivos de navegador:

```json5
{
  gateway: {
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["operator@example.com"],
        deviceAutoApprove: {
          enabled: true,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

El valor predeterminado es `enabled: false`. Cuando se habilita, se aplican todas estas reglas:

1. El WebSocket debe haberse autenticado mediante el método `trusted-proxy` con una identidad de usuario no vacía que haya superado `allowUsers` cuando haya una lista de usuarios permitidos configurada. Las conexiones mediante token, contraseña, Tailscale y sin autenticar nunca utilizan esta política.
2. Solo se puede aprobar automáticamente un nuevo dispositivo de navegador de la interfaz de control o WebChat. Cualquier solicitud para un dispositivo existente, incluida una ampliación de ámbitos, permanece pendiente de aprobación manual mediante `openclaw devices approve <requestId>`.
3. El dispositivo se aprueba con el rol `operator`. Si la solicitud de conexión incluye ámbitos, la concesión corresponde a la intersección exacta entre los ámbitos solicitados y `deviceAutoApprove.scopes`. Si la solicitud omite los ámbitos, se concede la lista configurada; cuando se omite esa lista, los valores predeterminados son `operator.read`, `operator.write` y `operator.approvals`. A continuación, la concesión resultante también queda limitada por el encabezado de proxy [`x-openclaw-scopes`](#control-ui-pairing-behavior) de la conexión, si está presente, de modo que un proxy que restrinja los ámbitos de un usuario también limite la concesión **persistente** del dispositivo, no solo la sesión; un encabezado presente pero vacío no concede ningún ámbito. Este límite se aplica incluso cuando el cliente omite su propia lista de ámbitos.
4. `operator.admin` solo se permite si figura explícitamente en `deviceAutoApprove.scopes`. Cuando se incluye, cada usuario autenticado por el proxy puede solicitar y recibir automáticamente privilegios administrativos completos en un nuevo dispositivo de navegador; las solicitudes sin ámbitos reciben automáticamente privilegios administrativos completos. `openclaw security audit` informa del hallazgo CRÍTICO `gateway.trusted_proxy_device_auto_approve_admin`, y el Gateway registra una advertencia una vez durante el inicio. Es preferible realizar la aprobación administrativa manual mediante `openclaw devices approve` o `openclaw devices rotate` hasta que estén disponibles los roles por identidad.

<Warning>
Al habilitar esta opción, el registro de nuevos dispositivos de navegador se delega por completo en la identidad del proxy inverso. Una cuenta de proxy vulnerada puede registrar un dispositivo persistente con todos los ámbitos configurados. Incluir `operator.admin` convierte dicho dispositivo en administrador con privilegios completos sin aprobación manual. Mantenga el Gateway accesible únicamente a través del proxy, exija una autenticación robusta en el proxy, sobrescriba los encabezados de identidad y utilice una lista `allowUsers` restringida.
</Warning>

## Comportamiento del emparejamiento de la interfaz de control

Cuando `gateway.auth.mode = "trusted-proxy"` está activo y la solicitud supera las comprobaciones del proxy de confianza, las sesiones WebSocket de la interfaz de control pueden conectarse sin identidad de emparejamiento del dispositivo.

Implicaciones para los ámbitos:

- Las sesiones WebSocket de la interfaz de control sin dispositivo se conectan, pero de forma predeterminada no reciben ningún ámbito de operador. OpenClaw vacía la lista de ámbitos solicitados y la convierte en `[]` para que una sesión que no esté vinculada a un dispositivo o token emparejado y aprobado no pueda autodeclarar permisos.
- Si los métodos fallan con `missing scope` después de establecer correctamente una conexión WebSocket, utilice HTTPS para que el navegador pueda generar la identidad del dispositivo y completar el emparejamiento. Consulte [HTTP no seguro de la interfaz de control](/es/web/control-ui#insecure-http).
- Las configuraciones antiguas que todavía contienen la clave retirada
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` utilizan la
  [migración de actualización de la interfaz de control](/es/web/control-ui#device-pairing-first-connection) limitada.

Limitación de ámbitos mediante el proxy inverso: si el proxy envía `x-openclaw-scopes` en la solicitud de actualización a WebSocket de la interfaz de control, OpenClaw limita los ámbitos de la sesión a la intersección entre los ámbitos solicitados y los declarados. Este encabezado no concede ámbitos; únicamente restringe los que puede tener la sesión. Cuando `deviceAutoApprove.enabled` es verdadero, el mismo límite también se aplica a la concesión persistente del dispositivo escrita por la [aprobación automática de dispositivos](#automatic-device-approval), de modo que un dispositivo aprobado automáticamente nunca tenga más ámbitos que los declarados por el proxy.

Implicaciones:

- El emparejamiento deja de ser el control principal para el acceso a la interfaz de control sin dispositivo. Cuando `deviceAutoApprove.enabled` es verdadero, la identidad del proxy también se convierte en el control de aprobación para registrar nuevos dispositivos de navegador.
- La política de autenticación del proxy inverso y `allowUsers` se convierten en el control de acceso efectivo.
- Mantenga el acceso de entrada al Gateway restringido únicamente a las IP de proxy de confianza (`gateway.trustedProxies` + cortafuegos).

Los clientes WebSocket personalizados no son sesiones de la interfaz de control. La entrada de actualización retirada de la interfaz de control no concede acceso temporal a clientes arbitrarios
`client.mode: "backend"` ni con formato de CLI. La automatización personalizada debe utilizar
la identidad y el emparejamiento de dispositivos, la ruta auxiliar reservada del backend local directo `client.id: "gateway-client"`
o el [Plugin RPC HTTP de administración](/es/plugins/admin-http-rpc)
cuando sea más adecuada una interfaz HTTP de solicitud y respuesta.

## Encabezado de ámbitos del operador

La autenticación mediante proxy de confianza es un modo HTTP **que contiene identidad**, por lo que los clientes pueden declarar opcionalmente ámbitos de operador con `x-openclaw-scopes` en las solicitudes a la API HTTP.

Nota: los ámbitos de WebSocket se determinan mediante el protocolo de enlace del Gateway y la vinculación de la identidad del dispositivo. En las solicitudes de actualización a WebSocket de la interfaz de control, `x-openclaw-scopes` solo limita los ámbitos negociados de la sesión, no los concede. Consulte el [comportamiento de emparejamiento de la interfaz de control](#control-ui-pairing-behavior).

Ejemplos:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

Comportamiento:

- Cuando el encabezado está presente, OpenClaw respeta el conjunto de ámbitos declarado.
- Cuando el encabezado está presente pero vacío, la solicitud declara que no tiene **ningún** ámbito de operador.
- Cuando el encabezado está ausente, las API HTTP normales que contienen identidad recurren al conjunto estándar de ámbitos predeterminados del operador (`operator.admin`, `operator.read`, `operator.write`, `operator.approvals`, `operator.pairing`, `operator.talk.secrets`).
- Las **rutas HTTP de plugins** con autenticación del Gateway son más restrictivas de forma predeterminada: cuando `x-openclaw-scopes` está ausente, su ámbito de ejecución recurre únicamente a `operator.write`.
- Las solicitudes HTTP con origen en el navegador deben seguir superando `gateway.controlUi.allowedOrigins` (o el modo alternativo deliberado mediante el encabezado Host), incluso después de que la autenticación mediante proxy de confianza se complete correctamente.

Regla práctica: envíe `x-openclaw-scopes` explícitamente cuando quiera que una solicitud mediante proxy de confianza sea más restrictiva que los valores predeterminados o cuando una ruta de plugin con autenticación del Gateway necesite algo más potente que el ámbito de escritura.

## Terminación TLS y HSTS

Use un único punto de terminación TLS y aplique HSTS allí.

<Tabs>
  <Tab title="Terminación TLS en el proxy (recomendada)">
    Cuando el proxy inverso gestiona HTTPS para `https://control.example.com`, configure `Strict-Transport-Security` en el proxy para ese dominio.

    - Adecuado para implementaciones expuestas a Internet.
    - Mantiene la política de certificados y protección de HTTP en un solo lugar.
    - OpenClaw puede permanecer en HTTP de bucle invertido detrás del proxy.

    Valor de encabezado de ejemplo:

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="Terminación TLS en el Gateway">
    Si OpenClaw sirve HTTPS directamente (sin un proxy que termine TLS), configure:

    ```json5
    {
      gateway: {
        tls: { enabled: true },
        http: {
          securityHeaders: {
            strictTransportSecurity: "max-age=31536000; includeSubDomains",
          },
        },
      },
    }
    ```

    `strictTransportSecurity` acepta un valor de encabezado de cadena o `false` para deshabilitarlo explícitamente.

  </Tab>
</Tabs>

### Orientación para el despliegue

- Comience primero con una antigüedad máxima corta (por ejemplo, `max-age=300`) mientras valida el tráfico.
- Aumente a valores de larga duración (por ejemplo, `max-age=31536000`) solo cuando tenga un alto grado de confianza.
- Añada `includeSubDomains` solo si todos los subdominios están preparados para HTTPS.
- Use la precarga solo si cumple deliberadamente los requisitos de precarga para todo el conjunto de dominios.
- El desarrollo local limitado al bucle invertido no se beneficia de HSTS.

## Ejemplos de configuración del proxy

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium transmite la identidad en `x-pomerium-claim-email` (u otros encabezados de declaraciones) y un JWT en `x-pomerium-jwt-assertion`.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP de Pomerium
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-pomerium-claim-email",
            requiredHeaders: ["x-pomerium-jwt-assertion"],
          },
        },
      },
    }
    ```

    Fragmento de configuración de Pomerium:

    ```yaml
    routes:
      - from: https://openclaw.example.com
        to: http://openclaw-gateway:18789
        policy:
          - allow:
              or:
                - email:
                    is: nick@example.com
        pass_identity_headers: true
    ```

  </Accordion>
  <Accordion title="Caddy con OAuth">
    Caddy puede autenticar usuarios con el plugin `caddy-security` y transmitir encabezados de identidad.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP de Caddy/proxy auxiliar
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    Fragmento de Caddyfile:

    ```caddy
    openclaw.example.com {
        authenticate with oauth2_provider
        authorize with policy1

        reverse_proxy openclaw:18789 {
            header_up X-Forwarded-User {http.auth.user.email}
        }
    }
    ```

  </Accordion>
  <Accordion title="nginx + oauth2-proxy">
    oauth2-proxy autentica a los usuarios y transmite la identidad en `x-auth-request-email`.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // IP de nginx/oauth2-proxy
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    Fragmento de configuración de nginx:

    ```nginx
    location / {
        auth_request /oauth2/auth;
        auth_request_set $user $upstream_http_x_auth_request_email;

        proxy_pass http://openclaw:18789;
        proxy_set_header X-Auth-Request-Email $user;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    ```

  </Accordion>
  <Accordion title="Traefik con autenticación reenviada">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // IP del contenedor de Traefik
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Configuración mixta de tokens

El inicio del Gateway rechaza la autenticación mediante proxy de confianza si también hay configurado un token compartido (`gateway.auth.token` o `OPENCLAW_GATEWAY_TOKEN`). Ambos son mutuamente excluyentes porque un token compartido permitiría a los clientes del mismo host autenticarse mediante una ruta completamente diferente de la identidad verificada por el proxy que este modo está diseñado para exigir.

Si el inicio falla con un error como `gateway auth mode is trusted-proxy, but a shared token is also configured`:

- Elimine el token compartido cuando use el modo de proxy de confianza, o
- Cambie `gateway.auth.mode` a `"token"` si pretende usar autenticación mediante tokens.

Los encabezados de identidad del proxy de confianza en el bucle invertido siguen produciendo un fallo seguro: los clientes del mismo host no se autentican silenciosamente como usuarios del proxy. En su lugar, los clientes internos de OpenClaw que omitan el proxy pueden autenticarse con `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`. El uso alternativo de tokens sigue sin admitirse de forma intencionada en el modo de proxy de confianza.

## Lista de comprobación de seguridad

Antes de habilitar la autenticación mediante proxy de confianza, verifique lo siguiente:

- [ ] **El proxy es la única ruta**: El puerto del Gateway está protegido mediante un cortafuegos frente a todo excepto el proxy.
- [ ] **trustedProxies es mínimo**: Solo incluye las IP reales de los proxies, no subredes completas.
- [ ] **El origen del proxy de bucle invertido es deliberado**: La autenticación mediante proxy de confianza produce un fallo seguro para las solicitudes originadas en el bucle invertido, a menos que `gateway.auth.trustedProxy.allowLoopback` se habilite explícitamente para un proxy del mismo host.
- [ ] **El proxy elimina los encabezados**: El proxy sobrescribe (no añade) los encabezados `x-forwarded-*` de los clientes.
- [ ] **Terminación TLS**: El proxy gestiona TLS; los usuarios se conectan mediante HTTPS.
- [ ] **allowedOrigins es explícito**: La interfaz de control fuera del bucle invertido usa `gateway.controlUi.allowedOrigins` explícito.
- [ ] **allowUsers está configurado** (recomendado): Restrinja el acceso a usuarios conocidos en lugar de permitir a cualquiera que esté autenticado.
- [ ] **No hay una configuración mixta de tokens**: No configure simultáneamente `gateway.auth.token` y `gateway.auth.mode: "trusted-proxy"`.
- [ ] **El uso alternativo de contraseñas locales es privado**: Si configura `gateway.auth.password` para clientes internos directos, mantenga el puerto del Gateway protegido mediante un cortafuegos para que los clientes remotos que no usen el proxy no puedan acceder directamente.
- [ ] **La aprobación automática de dispositivos es deliberada**: Si `deviceAutoApprove.enabled` es verdadero, trate la seguridad de la cuenta del proxy inverso como el límite de inscripción de dispositivos y mantenga la lista de ámbitos concedidos sin privilegios de administración y al mínimo.

## Auditoría de seguridad

`openclaw security audit` señala la autenticación mediante proxy de confianza con un hallazgo de gravedad **crítica**. Esto es intencionado; sirve para recordar que se está delegando la seguridad a la configuración del proxy.

La auditoría comprueba:

- Advertencia o recordatorio crítico de `gateway.trusted_proxy_auth` básico.
- Falta la configuración de `trustedProxies`.
- Falta la configuración de `userHeader`.
- `allowUsers` vacío (permite a cualquier usuario autenticado).
- `allowLoopback` habilitado para los orígenes de proxy del mismo host.
- Aprobación automática de dispositivos del navegador habilitada (delega el emparejamiento de nuevos dispositivos a la identidad del proxy).

También se aplican hallazgos independientes y no específicos del proxy de confianza cuando la interfaz de control está expuesta: `gateway.controlUi.allowedOrigins` con comodín o ausente y uso alternativo del origen mediante el encabezado Host.

## Solución de problemas

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    La solicitud no procedía de una IP incluida en `gateway.trustedProxies`. Compruebe:

    - ¿Es correcta la IP del proxy? (Las IP de los contenedores Docker pueden cambiar).
    - ¿Hay un equilibrador de carga delante del proxy?
    - Use `docker inspect` o `kubectl get pods -o wide` para encontrar las IP reales.

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw rechazó una solicitud de proxy de confianza con origen en el bucle invertido.

    Compruebe:

    - ¿El proxy se conecta desde `127.0.0.1` / `::1`?
    - ¿Está intentando usar la autenticación mediante proxy de confianza con un proxy inverso de bucle invertido en el mismo host?

    Solución:

    - Prefiera la autenticación mediante token o contraseña para los clientes internos del mismo host que no pasen por el proxy, o
    - Enrute a través de una dirección de proxy de confianza que no sea de bucle invertido y mantenga esa IP en `gateway.trustedProxies`, o
    - Para un proxy inverso deliberado en el mismo host, configure `gateway.auth.trustedProxy.allowLoopback = true`, mantenga la dirección de bucle invertido en `gateway.trustedProxies` y asegúrese de que el proxy elimine o sobrescriba los encabezados de identidad.

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    La IP de origen de la solicitud coincidía con una de las direcciones de las interfaces de red propias del host del Gateway que no son de bucle invertido (no con el proxy), como protección contra tráfico suplantado del mismo host en redes Tailscale o redes puente de Docker. `..._check_failed` significa que se produjo un error en la propia detección de interfaces, por lo que OpenClaw produce un fallo seguro.

    Compruebe:

    - ¿Algún proceso del propio host del Gateway envía directamente encabezados de identidad y omite el proxy?
    - ¿El proxy se ejecuta en el mismo espacio de nombres de red que el Gateway, con una IP que también aparece como interfaz local?

    Solución: enrute el tráfico del proxy mediante una dirección que no esté también vinculada localmente por el host del Gateway o use `allowLoopback` únicamente para una configuración real de proxy en el mismo host.

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    El encabezado del usuario estaba vacío o ausente. Compruebe:

    - ¿El proxy está configurado para transmitir encabezados de identidad?
    - ¿El nombre del encabezado es correcto? (No distingue entre mayúsculas y minúsculas, pero la ortografía es importante).
    - ¿El usuario está realmente autenticado en el proxy?

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    Un encabezado obligatorio no estaba presente. Compruebe:

    - La configuración del proxy para esos encabezados específicos.
    - Si los encabezados se están eliminando en algún punto de la cadena.

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    El usuario está autenticado, pero no está en `allowUsers`. Añádalo o elimine la lista de permitidos.
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode` es `"trusted-proxy"`, pero `gateway.trustedProxies` está vacío, o falta el propio `gateway.auth.trustedProxy`. Todas las solicitudes se rechazan hasta que ambos estén configurados.
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    La autenticación mediante proxy de confianza se realizó correctamente, pero el encabezado `Origin` del navegador no superó las comprobaciones de origen de la interfaz de control.

    Compruebe lo siguiente:

    - `gateway.controlUi.allowedOrigins` incluye el origen exacto del navegador.
    - No se depende de orígenes comodín, salvo que se desee intencionadamente permitir todos los orígenes.
    - Si se utiliza intencionadamente el modo alternativo basado en el encabezado Host, `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` está configurado deliberadamente.

  </Accordion>
  <Accordion title="La conexión se establece, pero los métodos indican que falta un ámbito">
    El WebSocket se conecta, pero `chat.history`, `sessions.list` o
    `models.list` falla con `missing scope: operator.read`.

    Causas habituales:

    - Sesión de la interfaz de control sin dispositivo: la autenticación mediante proxy de confianza puede permitir la conexión WebSocket sin identidad de dispositivo, pero OpenClaw elimina los ámbitos de las sesiones sin dispositivo por diseño.
    - Cliente de backend personalizado: la entrada retirada de actualización de la interfaz de control nunca concede acceso a clientes WebSocket arbitrarios con formato de backend o CLI.
    - `x-openclaw-scopes` demasiado restrictivo: si el proxy inserta este encabezado en la solicitud de actualización de WebSocket de la interfaz de control, los ámbitos de la sesión quedan limitados a ese conjunto. Un valor de encabezado vacío no concede ningún ámbito.

    Solución:

    - Para la interfaz de control, utilice HTTPS para que el navegador pueda generar una identidad de dispositivo y completar el emparejamiento.
    - Para la automatización personalizada, utilice identidad de dispositivo/emparejamiento, la ruta auxiliar de backend local directo reservada `gateway-client` o [RPC HTTP de administración](/es/plugins/admin-http-rpc).
    - No añada la clave retirada `gateway.controlUi.dangerouslyDisableDeviceAuth` a la configuración actual. Las instalaciones anteriores utilizan automáticamente la migración de autoemparejamiento de una sola vez.

  </Accordion>
  <Accordion title="WebSocket sigue fallando">
    Asegúrese de que el proxy:

    - Admite actualizaciones de WebSocket (`Upgrade: websocket`, `Connection: upgrade`).
    - Transmite los encabezados de identidad en las solicitudes de actualización de WebSocket (no solo en HTTP).
    - No tiene una ruta de autenticación independiente para las conexiones WebSocket.

  </Accordion>
</AccordionGroup>

## Migración desde la autenticación mediante token

<Steps>
  <Step title="Configurar el proxy">
    Configure el proxy para autenticar a los usuarios y transmitir encabezados.
  </Step>
  <Step title="Probar el proxy de forma independiente">
    Pruebe la configuración del proxy de forma independiente (curl con encabezados).
  </Step>
  <Step title="Actualizar la configuración de OpenClaw">
    Actualice la configuración de OpenClaw con autenticación mediante proxy de confianza.
  </Step>
  <Step title="Reiniciar el Gateway">
    Reinicie el Gateway.
  </Step>
  <Step title="Probar WebSocket">
    Pruebe las conexiones WebSocket desde la interfaz de control.
  </Step>
  <Step title="Auditar">
    Ejecute `openclaw security audit` y revise los resultados.
  </Step>
</Steps>

## Contenido relacionado

- [Configuración](/es/gateway/configuration) — referencia de configuración
- [Ámbitos del operador](/es/gateway/operator-scopes) — roles, ámbitos y comprobaciones de aprobación
- [Acceso remoto](/es/gateway/remote) — otros patrones de acceso remoto
- [Seguridad](/es/gateway/security) — guía de seguridad completa
- [Tailscale](/es/gateway/tailscale) — alternativa más sencilla para el acceso exclusivo desde la tailnet
