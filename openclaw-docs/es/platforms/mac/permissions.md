---
read_when:
    - Depuración de solicitudes de permisos de macOS que no aparecen o están bloqueadas
    - Decidir si se concede acceso de Accesibilidad a Node o a un entorno de ejecución de la CLI
    - Empaquetado o firma de la aplicación para macOS
    - Cambio de los ID de paquete o las rutas de instalación de la aplicación
summary: Persistencia de permisos de macOS (TCC) y requisitos de firma
title: Permisos de macOS
x-i18n:
    generated_at: "2026-07-26T05:19:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e561aa641e44fc1e1b95a3db244f31124e4e51d13ae709bee188d86054301e34
    source_path: platforms/mac/permissions.md
    workflow: 16
---

Los permisos concedidos en macOS son frágiles. TCC asocia un permiso concedido con la firma de código de la aplicación, el identificador del paquete y la ruta en disco. Si cualquiera de estos elementos cambia, macOS trata la aplicación como nueva y puede omitir u ocultar las solicitudes de permiso.

## Requisitos para permisos estables

- Misma ruta: ejecute la aplicación desde una ubicación fija (para OpenClaw, `dist/OpenClaw.app`).
- Mismo identificador del paquete: el ID de paquete de OpenClaw es `ai.openclaw.mac`; cambiarlo crea una nueva identidad de permisos.
- Aplicación firmada: las compilaciones sin firmar o con firma ad hoc no conservan los permisos.
- Firma coherente: use un certificado real de Apple Development o Developer ID para que la firma se mantenga estable entre recompilaciones.

Las firmas ad hoc generan una identidad nueva en cada compilación. macOS olvida los permisos concedidos anteriormente y las solicitudes pueden desaparecer por completo hasta que se eliminen las entradas obsoletas.

## Permisos de accesibilidad para entornos de ejecución de Node y CLI

Es preferible conceder Accesibilidad a OpenClaw.app, Peekaboo.app u otro asistente firmado con su propio identificador de paquete, en lugar de a un binario genérico de `node`.

TCC de macOS concede Accesibilidad a la identidad de código del proceso que detecta. Si un flujo de trabajo de Homebrew, nvm, pnpm o npm hace que un ejecutable compartido de `node` reciba Accesibilidad, cualquier paquete de JavaScript iniciado mediante ese mismo ejecutable puede heredar privilegios de automatización de la interfaz gráfica.

Considere una entrada de `node` en System Settings como un permiso amplio para ese entorno de ejecución de Node, no como un permiso para un único paquete de npm. Evite conceder Accesibilidad a `node` a menos que confíe en todos los scripts y paquetes iniciados mediante esa instalación exacta de Node.

La aprobación de Accesibilidad no habilita el uso compartido de actividad. **Settings -> Permissions -> Active computer detection** es un control independiente, desactivado de forma predeterminada, para compartir con el Gateway una duración de inactividad limitada. Al desactivarlo, se borra la actividad conservada sin revocar la Accesibilidad ni desconectar el Node.

Si concedió Accesibilidad accidentalmente a `node`, elimine esa entrada de System Settings -> Privacy & Security -> Accessibility. A continuación, conceda el permiso a la aplicación o al asistente firmado que deba encargarse de la automatización de la interfaz.

## Lista de comprobación para la recuperación cuando desaparecen las solicitudes

1. Cierre la aplicación.
2. Elimine la entrada de la aplicación en System Settings -> Privacy & Security.
3. Vuelva a iniciar la aplicación desde la misma ruta y conceda de nuevo los permisos.
4. Si la solicitud sigue sin aparecer, restablezca las entradas de TCC con `tccutil` e inténtelo de nuevo.
5. Algunos permisos solo vuelven a aparecer después de reiniciar macOS por completo.

Ejemplos de restablecimiento (con el ID de paquete de OpenClaw, `ai.openclaw.mac`):

```bash
sudo tccutil reset Accessibility ai.openclaw.mac
sudo tccutil reset ScreenCapture ai.openclaw.mac
sudo tccutil reset AppleEvents
```

## Permisos de archivos y carpetas (Desktop/Documents/Downloads)

macOS también puede restringir Desktop, Documents y Downloads para los procesos de terminal o en segundo plano. Si la lectura de archivos o la enumeración de directorios se bloquean, conceda acceso al mismo contexto de proceso que realiza las operaciones con archivos (por ejemplo, Terminal/iTerm, una aplicación iniciada mediante LaunchAgent o un proceso SSH).

Solución alternativa: mueva los archivos al espacio de trabajo de OpenClaw (`~/.openclaw/workspace`) si desea evitar conceder permisos para cada carpeta.

Si está probando permisos, firme siempre con un certificado real. Las compilaciones ad hoc solo son aceptables para ejecuciones locales rápidas en las que los permisos no sean importantes.

## Contenido relacionado

- [Aplicación para macOS](/es/platforms/macos)
- [Firma para macOS](/es/platforms/mac/signing)
