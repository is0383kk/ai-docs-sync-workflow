---
read_when:
    - Quieres conectar OpenClaw con WeChat o Weixin
    - Está instalando o solucionando problemas del Plugin de canal openclaw-weixin
    - Necesita comprender cómo se ejecutan los plugins de canales externos junto al Gateway
summary: Configuración del canal de WeChat mediante el plugin externo openclaw-weixin
title: WeChat
x-i18n:
    generated_at: "2026-07-26T04:31:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98faf95f9fb76deedb7df9adf3092083722a77bdd793de98c41a6f715cc0d14a
    source_path: channels/wechat.md
    workflow: 16
---

OpenClaw se conecta a WeChat mediante el Plugin externo de canal de Tencent
`@tencent-weixin/openclaw-weixin`.

Estado: Plugin externo, mantenido por el equipo de Tencent Weixin. Se admiten los chats directos y
el contenido multimedia. Los chats de grupo no se anuncian en los metadatos de capacidades
del Plugin (solo declara chats directos).

## Nomenclatura

- **WeChat** es el nombre que se muestra a los usuarios en esta documentación.
- **Weixin** es el nombre que usa el paquete de Tencent y el id. del Plugin.
- `openclaw-weixin` es el id. de canal de OpenClaw (`weixin` y `wechat` funcionan como alias).
- `@tencent-weixin/openclaw-weixin` es el paquete npm.

Use `openclaw-weixin` en los comandos de la CLI y las rutas de configuración.

## Funcionamiento

El código de WeChat no se encuentra en el repositorio principal de OpenClaw. OpenClaw proporciona el
contrato genérico de Plugin de canal, mientras que el Plugin externo proporciona el
entorno de ejecución específico de WeChat:

1. `openclaw plugins install` instala `@tencent-weixin/openclaw-weixin`.
2. El Gateway detecta el manifiesto del Plugin y carga su punto de entrada.
3. El Plugin registra el id. de canal `openclaw-weixin`.
4. `openclaw channels login --channel openclaw-weixin` inicia el acceso mediante QR.
5. El Plugin almacena las credenciales de la cuenta en el directorio de estado de OpenClaw
   (`~/.openclaw` de forma predeterminada).
6. Cuando se inicia el Gateway, el Plugin inicia su monitor de Weixin para cada
   cuenta configurada.
7. Los mensajes entrantes de WeChat se normalizan mediante el contrato de canal, se encaminan al
   agente de OpenClaw seleccionado y se devuelven mediante la ruta de salida del Plugin.

Esta separación es importante: el núcleo de OpenClaw permanece independiente de los canales. El acceso a WeChat,
las llamadas a la API iLink de Tencent, la carga y descarga de contenido multimedia, los tokens de contexto y la
supervisión de cuentas son responsabilidad del Plugin externo.

## Instalación

Instalación rápida:

```bash
npx -y @tencent-weixin/openclaw-weixin-cli install
```

Instalación manual:

```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw config set plugins.entries.openclaw-weixin.enabled true
```

Reinicie el Gateway después de la instalación:

```bash
openclaw gateway restart
```

## Inicio de sesión

Inicie el acceso mediante QR en el mismo equipo donde se ejecuta el Gateway:

```bash
openclaw channels login --channel openclaw-weixin
```

Escanee el código QR con WeChat en el teléfono y confirme el inicio de sesión. El Plugin guarda
localmente el token de la cuenta tras un escaneo correcto.

Para añadir otra cuenta de WeChat, vuelva a ejecutar el mismo comando de inicio de sesión. Si hay varias
cuentas, aísle las sesiones de mensajes directos por cuenta, canal y remitente:

```bash
openclaw config set session.dmScope per-account-channel-peer
```

## Control de acceso

Los mensajes directos utilizan el modelo normal de emparejamiento y lista de permitidos de OpenClaw para los
Plugins de canal.

Apruebe nuevos remitentes:

```bash
openclaw pairing list openclaw-weixin
openclaw pairing approve openclaw-weixin <CODE>
```

Para consultar el modelo completo de control de acceso, consulte [Emparejamiento](/es/channels/pairing).

## Compatibilidad

El Plugin comprueba la versión de OpenClaw del host durante el inicio.

| Línea del Plugin | Versión de OpenClaw                                             | Etiqueta npm |
| ----------- | --------------------------------------------------------------- | -------- |
| `2.x`       | `>=2026.5.12` (actualmente 2.4.6; las primeras versiones 2.x aceptaban `>=2026.3.22`) | `latest` |
| `1.x`       | `>=2026.1.0 <2026.3.22`                                         | `legacy` |

Si el Plugin indica que la versión de OpenClaw es demasiado antigua, actualice
OpenClaw o instale la línea heredada del Plugin:

```bash
openclaw plugins install @tencent-weixin/openclaw-weixin@legacy
```

## Proceso auxiliar

El Plugin de WeChat puede ejecutar tareas auxiliares junto al Gateway mientras supervisa la
API iLink de Tencent. En la incidencia #68451, esa ruta auxiliar puso de manifiesto un error en la
limpieza genérica de instancias obsoletas del Gateway de OpenClaw: un proceso secundario podía intentar limpiar el proceso
principal del Gateway, lo que provocaba bucles de reinicio con gestores de procesos como systemd.

La limpieza actual durante el inicio de OpenClaw excluye el proceso actual y sus antecesores,
por lo que un proceso auxiliar de canal no puede finalizar el Gateway que lo inició. Esta corrección es
genérica; no es una ruta específica de WeChat en el núcleo.

## Solución de problemas

Compruebe la instalación y el estado:

```bash
openclaw plugins list
openclaw channels status --probe
openclaw --version
```

Si el canal aparece como instalado, pero no se conecta, confirme que el Plugin esté
habilitado y reinicie:

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled true
openclaw gateway restart
```

Si el Gateway se reinicia repetidamente después de habilitar WeChat, actualice tanto OpenClaw como
el Plugin:

```bash
npm view @tencent-weixin/openclaw-weixin version
openclaw plugins install "@tencent-weixin/openclaw-weixin" --force
openclaw gateway restart
```

Si durante el inicio se informa de que el paquete del Plugin instalado `requires compiled runtime
output for TypeScript entry`, el paquete npm se publicó sin los archivos compilados
del entorno de ejecución de JavaScript que OpenClaw necesita. Actualice o vuelva a instalar el Plugin después de que su
editor publique un paquete corregido, o deshabilítelo o desinstálelo temporalmente.

Deshabilitación temporal:

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled false
openclaw gateway restart
```

## Documentación relacionada

- Descripción general de los canales: [Canales de chat](/es/channels)
- Emparejamiento: [Emparejamiento](/es/channels/pairing)
- Enrutamiento de canales: [Enrutamiento de canales](/es/channels/channel-routing)
- Arquitectura de Plugins: [Arquitectura de Plugins](/es/plugins/architecture)
- SDK de Plugins de canal: [SDK de Plugins de canal](/es/plugins/sdk-channel-plugins)
- Paquete externo: [@tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)
