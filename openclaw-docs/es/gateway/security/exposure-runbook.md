---
read_when:
    - Exposición del Gateway mediante LAN, tailnet, Tailscale Serve, Funnel o un proxy inverso
    - Revisión de un despliegue antes de permitir el acceso de usuarios reales de mensajería
    - Revertir una configuración arriesgada de acceso remoto o de mensajes directos
sidebarTitle: Exposure runbook
summary: Lista de comprobación previa y de reversión antes de exponer un Gateway de OpenClaw más allá de la interfaz de bucle local
title: Manual de exposición del Gateway
x-i18n:
    generated_at: "2026-07-26T04:41:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fb8e66af57e804325afc91281122b822183337177c734efe065c5fc18b175e72
    source_path: gateway/security/exposure-runbook.md
    workflow: 16
---

<Warning>
Exponga el Gateway solo después de poder explicar quién puede acceder a él, cómo se
autentica, qué agentes puede activar y qué herramientas pueden usar esos agentes.
En caso de duda, vuelva al acceso exclusivo mediante loopback y ejecute de nuevo la auditoría.
</Warning>

Este manual operativo convierte las directrices generales de [seguridad](/es/gateway/security) en una
lista de verificación para operadores sobre la exposición del acceso remoto y la mensajería.

## Elegir el patrón de exposición

Prefiera el patrón más restrictivo que satisfaga el flujo de trabajo.

| Patrón                     | Recomendado cuando                                     | Controles obligatorios                                                                                                                                        |
| -------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Loopback + túnel SSH       | Uso personal, acceso administrativo, depuración        | Mantenga `gateway.bind: "loopback"` y establezca el túnel para `127.0.0.1:18789`                                                                                     |
| Loopback + Tailscale Serve | Acceso desde una tailnet personal a la interfaz de control/WebSocket | Mantenga el Gateway accesible solo mediante loopback; los encabezados de identidad de Tailscale solo autentican la superficie WebSocket de la interfaz de control, no otras rutas de autenticación |
| Enlace a tailnet/LAN       | Red privada dedicada con dispositivos conocidos        | Autenticación del Gateway, lista de permitidos del firewall, sin reenvío de puertos públicos                                                                  |
| Proxy inverso de confianza | SSO/OIDC de la organización delante del Gateway        | Autenticación `trusted-proxy`, `trustedProxies` estricta, reglas para sobrescribir/eliminar encabezados, usuarios permitidos explícitos                   |
| Internet público           | Implementaciones excepcionales y de alto riesgo        | Proxy con reconocimiento de identidad, TLS, límites de frecuencia, listas de permitidos estrictas, sesiones secundarias aisladas                              |

Evite el reenvío directo de puertos públicos al Gateway. Si se requiere acceso
público, coloque delante un proxy con reconocimiento de identidad y haga que el
proxy sea la única ruta de red al Gateway.

## Inventario previo

Registre lo siguiente antes de cambiar la política de enlace, proxy, Tailscale o canal:

- Host, usuario del sistema operativo y directorio de estado del Gateway (valor predeterminado: `~/.openclaw`).
- URL y modo de enlace del Gateway (`gateway.bind`; puerto predeterminado: `18789`).
- Modo de autenticación, origen del token o la contraseña, u origen de identidad del proxy de confianza.
- Cada canal habilitado y si acepta mensajes directos, grupos o webhooks.
- Agentes accesibles para remitentes no locales.
- Perfil de herramientas, modo de aislamiento y política de herramientas con privilegios elevados de cada agente accesible.
- Credenciales externas disponibles para esos agentes.
- Ubicación de la copia de seguridad de `~/.openclaw/openclaw.json` y de las credenciales.

Si más de una persona puede enviar mensajes al bot, trátelo como autoridad
delegada compartida sobre las herramientas, no como aislamiento del host por usuario.

## Comprobaciones de referencia

Ejecútelas antes de habilitar el acceso:

```bash
openclaw doctor
openclaw security audit
openclaw security audit --deep
openclaw health
```

Resuelva primero los hallazgos críticos. Acepte advertencias solo cuando sean
intencionadas y estén documentadas para la implementación. Consulte [Comprobaciones de auditoría de seguridad](/es/gateway/security/audit-checks)
para saber qué significa cada `checkId` y cuál es su clave de corrección.

Para validar la CLI de forma remota, proporcione las credenciales explícitamente:

```bash
openclaw gateway probe --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

No presuponga que las credenciales de la configuración local se aplican a una URL remota explícita.

## Referencia mínima segura

Use esta estructura como punto de partida para implementaciones expuestas:

```json5
{
  gateway: {
    bind: "loopback",
    auth: {
      mode: "token",
      token: "replace-with-a-long-random-token",
    },
  },
  session: {
    dmScope: "per-channel-peer",
  },
  agents: {
    defaults: {
      sandbox: { mode: "non-main" },
    },
  },
  tools: {
    profile: "messaging",
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

Amplíe un control cada vez: añada una lista de permitidos para un canal específico
antes de habilitar herramientas con capacidad de escritura, o habilite un proxy
inverso antes de aceptar tráfico remoto de la interfaz de control.

`tools.exec.security: "deny"` bloquea todas las llamadas de ejecución, incluidos los
diagnósticos inofensivos. Si se requieren diagnósticos o comandos de bajo riesgo,
relaje este control únicamente después de elegir los remitentes, agentes, comandos
y el modo de aprobación específicos que correspondan a su modelo de amenazas.

## Exposición de mensajes directos y grupos

Los canales de mensajería son superficies de entrada no confiables. Antes de permitir
mensajes directos o grupos:

- Prefiera `dmPolicy: "pairing"` o una lista `allowFrom` estricta en lugar de `dmPolicy: "open"`.
- No combine listas de permitidos `"*"` con un acceso amplio a herramientas.
- Exija menciones en los grupos, a menos que la sala esté estrictamente controlada.
- Configure `session.dmScope: "per-channel-peer"` (o `"per-account-channel-peer"` para
  canales con varias cuentas) cuando varias personas puedan enviar mensajes directos al bot, para que las sesiones
  de mensajes directos no compartan contexto.
- Dirija los canales compartidos a agentes con herramientas mínimas y sin
  credenciales personales.

El emparejamiento autoriza al remitente a activar el bot. No convierte a ese
remitente en un límite de seguridad del host independiente.

## Comprobaciones del proxy inverso

Para proxies con reconocimiento de identidad:

- El proxy debe autenticar a los usuarios antes de reenviar tráfico al Gateway.
- El firewall o la política de red deben bloquear el acceso directo al puerto del Gateway.
- `gateway.trustedProxies` debe incluir únicamente las direcciones IP de origen del proxy.
- El proxy debe eliminar o sobrescribir los encabezados de identidad y reenvío
  proporcionados por el cliente.
- Configure `gateway.auth.trustedProxy.allowUsers` cuando el proxy atienda a más de
  una audiencia.
- Use `gateway.auth.trustedProxy.allowLoopback` solo para un proxy en el mismo host
  donde se confíe en los procesos locales y el proxy controle los encabezados de identidad.

Ejecute `openclaw security audit --deep` después de realizar cambios en el proxy. Los
hallazgos relacionados con proxies de confianza son especialmente relevantes porque el proxy se convierte
en el límite de autenticación.

## Revisión de herramientas y aislamiento

Antes de exponer un agente a remitentes remotos:

- Confirme qué sesiones se ejecutan en el host y cuáles en el entorno aislado.
- Deniegue la ejecución en el host o exija aprobación para ella.
- Mantenga deshabilitadas las herramientas con privilegios elevados, salvo que las necesite un remitente específico y de confianza.
- Evite las herramientas de navegador, lienzo, Node, Cron, Gateway y creación de sesiones en superficies de mensajería
  abiertas o parcialmente abiertas.
- Mantenga restringidos los montajes enlazados; evite las rutas de credenciales, del directorio personal, del socket de Docker y del sistema.
- Use gateways, usuarios del sistema operativo o hosts independientes para límites de confianza
  sustancialmente diferentes.

Si no se confía plenamente en los usuarios remotos, el aislamiento debe proceder de
implementaciones independientes, no solo de prompts o etiquetas de sesión.

## Validación posterior a los cambios

Después de cada cambio de exposición:

1. Ejecute de nuevo `openclaw security audit --deep`.
2. Confirme que se establece correctamente una conexión autorizada.
3. Confirme que se rechaza a un remitente o una sesión de navegador no autorizados.
4. Confirme que los registros ocultan los secretos.
5. Confirme que el enrutamiento de mensajes directos y grupos llega únicamente al agente previsto.
6. Confirme que las herramientas de alto impacto solicitan aprobación o se deniegan.
7. Documente las advertencias residuales aceptadas.

No continúe con el siguiente cambio de exposición hasta comprender el actual.

## Plan de reversión

Si el Gateway puede estar excesivamente expuesto:

```json5
{
  gateway: {
    bind: "loopback",
  },
  channels: {
    whatsapp: { dmPolicy: "disabled" },
    telegram: { dmPolicy: "disabled" },
    discord: { dmPolicy: "disabled" },
    slack: { dmPolicy: "disabled" },
  },
  tools: {
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false },
  },
}
```

A continuación:

1. Detenga el reenvío público, Tailscale Funnel o las rutas del proxy inverso.
2. Rote los tokens o contraseñas del Gateway y las credenciales de integración afectadas.
3. Elimine `"*"` y los remitentes inesperados de las listas de permitidos.
4. Revise los registros de auditoría recientes, el historial de ejecuciones, las llamadas a herramientas y los cambios de configuración.
5. Ejecute de nuevo `openclaw security audit --deep`.
6. Vuelva a habilitar el acceso con el patrón más restrictivo que satisfaga el flujo de trabajo.

## Lista de verificación de la revisión

- El Gateway sigue siendo accesible solo mediante loopback, salvo que exista un motivo documentado.
- El acceso que no usa loopback cuenta con autenticación y firewall, y no dispone de ninguna ruta pública directa.
- Las implementaciones con proxies de confianza tienen direcciones IP de proxy estrictas y controles de encabezados.
- Los mensajes directos usan emparejamiento o listas de permitidos, no acceso abierto de forma predeterminada.
- Los grupos exigen menciones o listas de permitidos explícitas.
- Los canales compartidos no pueden acceder a credenciales personales.
- Las sesiones secundarias se ejecutan en modo aislado.
- La ejecución en el host y las herramientas con privilegios elevados se deniegan o requieren aprobación.
- Los registros ocultan los secretos.
- Los hallazgos críticos de la auditoría están resueltos.
- Los pasos de reversión están probados y documentados.
