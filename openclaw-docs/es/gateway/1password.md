---
read_when:
    - Quieres que las claves de API no estén en openclaw.json, sino en 1Password
    - Se ejecuta el Gateway sin interfaz gráfica y se necesita autenticación de cuenta de servicio para op
    - Se desea que los agentes lean o inyecten secretos con la CLI `op`
summary: Resuelve los secretos del Gateway con la CLI de 1Password y permite que los agentes usen la skill 1password incluida
title: 1Password
x-i18n:
    generated_at: "2026-07-26T05:07:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb14944f0b3ce1ee3f90bf666a53e8673e7a9861e3e138a5fabe9c8e070cbd7
    source_path: gateway/1password.md
    workflow: 16
---

OpenClaw se integra con **1Password** de tres maneras independientes:

- **Secretos de configuración:** cualquier campo [SecretRef](/es/gateway/secrets) de `openclaw.json` puede resolverse mediante la CLI `op` en tiempo de ejecución, por lo que las claves de API nunca se almacenan en el archivo de configuración.
- **Flujos de trabajo de agentes:** la skill `1password` incluida enseña a los agentes a iniciar sesión y a leer o inyectar secretos con `op` para sus propias tareas.
- **Inicio de sesión en el navegador:** el backend `claude-cli` puede usar la integración de Chrome de Claude Code con [1Password para Claude](https://support.1password.com/1password-claude/), lo que permite al agente iniciar sesión en sitios web sin que la contraseña llegue al modelo ni a OpenClaw.

## Requisitos

- La [CLI de 1Password](https://developer.1password.com/docs/cli/get-started/) (`op`) instalada en el host del Gateway (`brew install 1password-cli` en macOS).
- Un modo de autenticación para `op`:
  - **Cuenta de servicio** (recomendada para Gateways sin interfaz gráfica): exporte `OP_SERVICE_ACCOUNT_TOKEN` en el entorno del servicio del Gateway. No requiere aplicación de escritorio ni inicio de sesión interactivo.
  - **Integración con la aplicación de escritorio**: la aplicación 1Password se ejecuta en el mismo equipo con la integración de la CLI habilitada. Las primeras llamadas pueden activar Touch ID o la autenticación del sistema.
  - **Inicio de sesión independiente**: `op signin` solicita autenticación en cada sesión. Es viable para agentes mediante la skill, pero no resulta adecuado para resolver secretos de configuración en un Gateway sin interfaz gráfica.

## Resolver secretos de configuración con op

Declare un proveedor de secretos de ejecución que ejecute `op read` con una referencia `op://vault/item/field` y, a continuación, dirija a él cualquier campo compatible con SecretRef:

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // required for Homebrew symlinked binaries
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

Cómo encajan las piezas:

- `command` debe ser una ruta absoluta; `trustedDirs` marca su directorio como de confianza y `allowSymlinkCommand` es necesario porque Homebrew instala `op` como un enlace simbólico.
- `args` transmite literalmente la referencia `op://vault/item/field`. OpenClaw no analiza por sí mismo el esquema `op://`; el binario `op` lo resuelve.
- `passEnv` reenvía las variables indicadas desde el entorno del Gateway. La integración con la aplicación de escritorio necesita `HOME`; las cuentas de servicio también necesitan que `OP_SERVICE_ACCOUNT_TOKEN` esté presente en el entorno del servicio del Gateway (añádalo a `passEnv` o establézcalo mediante `env` únicamente si acepta que el token pueda leerse en el archivo de configuración).
- Para una salida de un único valor, mantenga `id: "value"`. Con `jsonOnly: true` y una carga útil JSON, acceda a los campos mediante un identificador de puntero JSON.
- Una entrada de proveedor por secreto permite auditar las referencias; asigne a los proveedores nombres basados en sus consumidores (`onepassword_openai`, `onepassword_telegram`).

Consulte [Secretos del Gateway](/es/gateway/secrets) para conocer el orden de resolución, el almacenamiento en caché y la semántica de los errores, y [Superficie de credenciales SecretRef](/es/reference/secretref-credential-surface) para ver todos los campos que aceptan SecretRefs.

## Configuración de cuentas de servicio para Gateways sin interfaz gráfica

1. Cree una cuenta de servicio en su cuenta de 1Password y concédale acceso de lectura únicamente a los elementos de la bóveda que necesite el Gateway.
2. Proporcione `OP_SERVICE_ACCOUNT_TOKEN` al servicio del Gateway (plist de launchd, unidad de systemd o entorno del contenedor).
3. Añada `"OP_SERVICE_ACCOUNT_TOKEN"` a la lista `passEnv` del proveedor.
4. Verifique desde el entorno del host del Gateway: `op whoami` debería mostrar la cuenta de servicio sin solicitar autenticación.

Las lecturas de cuentas de servicio requieren que la bóveda se indique explícitamente en la referencia `op://`. Restrinja cuidadosamente el alcance de la cuenta; se trata de una credencial al portador.

## La skill 1password para agentes

OpenClaw incluye una skill `1password` que convierte a los agentes en operadores competentes de `op`: detecta el modo de autenticación disponible (cuenta de servicio, integración con la aplicación de escritorio o inicio de sesión independiente), verifica el acceso con `op whoami` antes de leer nada y prioriza `op run` / `op inject` frente a escribir valores secretos en el disco. La skill requiere el binario `op` y ofrece instalarlo mediante Homebrew cuando no está disponible.

Los agentes la usan para sus propios flujos de trabajo, por ejemplo, para leer un token de despliegue durante una tarea o inyectar variables de entorno en un comando. Es independiente de la resolución de secretos de configuración; el Gateway resuelve las SecretRefs sin intervención de ninguna skill.

## Inicio de sesión en el navegador con 1Password para Claude

[1Password para Claude](https://support.1password.com/1password-claude/) permite que Claude solicite un inicio de sesión mientras la extensión de navegador de 1Password rellena la credencial directamente en la página mediante un canal cifrado. El secreto nunca entra en el contexto del modelo, la transcripción ni OpenClaw. Cuando OpenClaw ejecuta el [backend `claude-cli`](/es/gateway/cli-backends#claude-cli-specifics) con la integración de Chrome de Claude Code habilitada, las tareas de los agentes pueden usar este flujo para sitios web que requieran una sesión real iniciada.

Además del propio backend, esto requiere:

- Un host de Gateway con macOS y Chrome, la [extensión Claude in Chrome](https://code.claude.com/docs/en/chrome) conectada, la aplicación de escritorio de 1Password y la extensión de navegador de 1Password (ambas en la versión 8.12.28 o posterior).
- Claude Code con una sesión iniciada en un plan directo de Anthropic (Pro, Max, Team o Enterprise). La integración de Chrome no está disponible mediante Amazon Bedrock, Google Cloud ni otros proveedores externos.
- La conexión inicial de 1Password en el lado de Anthropic: 1Password para Claude se configura mediante la aplicación de escritorio de Claude o el flujo de la extensión descrito en la [guía de 1Password](https://support.1password.com/1password-claude/), y actualmente se encuentra en fase beta para macOS. En 1Password Business, un administrador debe habilitar primero "Allow AI agents to autofill for users" en Policies; los planes Anthropic Team/Enterprise también incluyen la integración desactivada hasta que un Owner la habilite.
- Un [plugin de backend de CLI](/es/plugins/cli-backend-plugins) que añada `--chrome` a los argumentos de inicio de Claude; el backend incluido no habilita Chrome.
- Una persona en el host del Gateway: cada uso de credenciales muestra allí una solicitud de 1Password que debe confirmarse (por ejemplo, mediante Touch ID). Con una política de ejecución restrictiva, las propias llamadas a herramientas del navegador también se reenvían primero a su canal como aprobaciones de OpenClaw.

Antes de conectar esta integración con OpenClaw, verifique los componentes en una sesión interactiva en el host del Gateway: ejecute `claude --chrome`, confirme que la extensión se conecta y compruebe que las herramientas `claude-in-chrome` incluyen las herramientas de credenciales. Si no aparecen allí, tampoco aparecerán mediante OpenClaw.

1Password rellena los códigos de acceso de un solo uso en la misma página; nunca transmita códigos de verificación ni contraseñas por el chat. Actualmente, los Gateways sin interfaz gráfica o remotos no pueden usar este flujo porque tanto la aprobación como el navegador se encuentran en el host del Gateway.

## Notas de seguridad

- Los valores secretos resueltos mediante proveedores de ejecución permanecen en la memoria del Gateway; las instantáneas de configuración y las respuestas `config.get` ocultan los campos SecretRef.
- Nunca incluya valores secretos en `openclaw.json`, registros ni chats. Mantenga los nombres de los elementos en la configuración y los valores en 1Password.
- El registro de auditoría de 1Password muestra cada lectura de las cuentas de servicio, lo que facilita la rotación de claves y la revisión de incidentes.

## Solución de problemas

- `command not found` o errores de creación de procesos: use la ruta absoluta de `op` e incluya su directorio en `trustedDirs`.
- `op` se resuelve, pero las lecturas fallan con errores de enlaces simbólicos: establezca `allowSymlinkCommand: true` para las instalaciones de Homebrew.
- `account is not signed in`: para las cuentas de servicio, confirme que `OP_SERVICE_ACCOUNT_TOKEN` llega al servicio del Gateway y figura en `passEnv`; para la integración con la aplicación de escritorio, confirme que la aplicación está en ejecución y desbloqueada.
- Primeras lecturas lentas: aumente `timeoutMs` en el proveedor; los arranques en frío de `op` pueden superar los tiempos de espera estrictos en hosts con mucha carga.
