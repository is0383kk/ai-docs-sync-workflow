---
read_when:
    - Actualización de una instalación existente de Matrix
    - Migración del historial cifrado y del estado de los dispositivos de Matrix
summary: Cómo OpenClaw actualiza el Plugin anterior de Matrix en el mismo lugar, incluidos los límites de recuperación del estado cifrado y los pasos de recuperación manual.
title: Migración de Matrix
x-i18n:
    generated_at: "2026-07-26T04:30:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 475c96914900a5597f37001264bd3d8f69a69dbd0600f2704c2a1be46924fac4
    source_path: channels/matrix-migration.md
    workflow: 16
---

Actualiza desde el plugin público anterior `matrix` a la implementación actual.

Para la mayoría de los usuarios, la actualización se realiza sin cambios:

- el plugin sigue siendo `@openclaw/matrix`
- el canal sigue siendo `matrix`
- la configuración permanece en `channels.matrix`
- las credenciales almacenadas en caché se trasladan al estado compartido del plugin `state/openclaw.sqlite`
- el estado de ejecución permanece en `~/.openclaw/matrix/`

No es necesario cambiar el nombre de las claves de configuración ni reinstalar el plugin con un nombre nuevo.
El paquete raíz `openclaw` ya no incluye el código de ejecución de Matrix ni las dependencias del SDK de Matrix. Si `openclaw channels status` muestra que Matrix está configurado, pero el plugin no está instalado, ejecuta `openclaw doctor --fix` o `openclaw plugins install @openclaw/matrix`; no instales paquetes del SDK de Matrix en el paquete raíz de OpenClaw.

## Qué hace la migración automáticamente

La migración de Matrix se ejecuta al ejecutar [`openclaw doctor --fix`](/es/gateway/doctor). Los archivos auxiliares basados en archivos junto al almacén dedicado de Matrix conservan su alternativa al iniciar el cliente, pero la importación de archivos de credenciales solo se realiza mediante Doctor; durante la ejecución, solo se lee el estado canónico de las credenciales en SQLite.

La migración de Doctor abarca:

- la importación y verificación de los archivos retirados `~/.openclaw/credentials/matrix/credentials*.json` antes de archivarlos
- el mantenimiento de la misma selección de cuenta y configuración `channels.matrix`
- la importación del estado auxiliar basado en archivos (caché de sincronización `bot-storage.json`, `recovery-key.json`, `legacy-crypto-migration.json`, instantáneas de IndexedDB) al estado SQLite de Matrix; los archivos migrados se archivan con el sufijo `.migrated`
- la reutilización de la raíz de almacenamiento de hashes de tokens existente más completa para la misma cuenta, servidor de origen, usuario y dispositivo de Matrix cuando el token de acceso cambia posteriormente

## Actualización desde versiones de OpenClaw anteriores a 2026.4

Las versiones hasta la serie 2026.6 también migraban el diseño plano original de almacén único de Matrix (`~/.openclaw/matrix/bot-storage.json` más `~/.openclaw/matrix/crypto/`) y preparaban la recuperación del estado cifrado desde el antiguo almacén criptográfico de Rust. Las versiones actuales ya no incluyen esa migración.

Si se actualiza una instalación que todavía utiliza el diseño plano, primero hay que actualizar a una versión 2026.6, ejecutar `openclaw doctor --fix` e iniciar el Gateway una vez para que se migren el almacén plano y las claves de sala recuperables. Después, se puede actualizar a la versión más reciente.

El plugin público anterior de Matrix **no** creaba automáticamente copias de seguridad de las claves de sala de Matrix. Si la instalación anterior contenía un historial cifrado únicamente de forma local que nunca se guardó en una copia de seguridad, es posible que algunos mensajes cifrados antiguos sigan siendo ilegibles tras la actualización, independientemente de la ruta de migración.

## Flujo de actualización recomendado

1. Actualiza OpenClaw y el plugin de Matrix de la forma habitual.
2. Ejecuta:

   ```bash
   openclaw doctor --fix
   ```

3. Inicia o reinicia el Gateway.
4. Comprueba el estado actual de la verificación y de la copia de seguridad:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. Coloca la clave de recuperación de la cuenta de Matrix que se está reparando en una variable de entorno específica de la cuenta. Para una única cuenta predeterminada, `MATRIX_RECOVERY_KEY` es suficiente. Para varias cuentas, utiliza una variable por cuenta, por ejemplo, `MATRIX_RECOVERY_KEY_ASSISTANT`, y añade `--account assistant` al comando.

6. Si OpenClaw indica que se necesita una clave de recuperación, ejecuta el comando correspondiente a la cuenta:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify backup restore --recovery-key-stdin --account assistant
   ```

7. Si este dispositivo sigue sin verificar, ejecuta el comando correspondiente a la cuenta:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify device --recovery-key-stdin --account assistant
   ```

   Si se acepta la clave de recuperación y la copia de seguridad se puede utilizar, pero `Cross-signing verified` sigue siendo `no`, completa la autoverificación desde otro cliente de Matrix:

   ```bash
   openclaw matrix verify self
   ```

   Acepta la solicitud en otro cliente de Matrix, compara los emojis o los números decimales y escribe `yes` solo cuando coincidan. El comando espera a que la identidad de Matrix sea de plena confianza antes de indicar que la operación se ha realizado correctamente.

8. Si se va a abandonar deliberadamente el historial antiguo irrecuperable y se desea una nueva base de referencia de copia de seguridad para los mensajes futuros, ejecuta:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

   Añade `--rotate-recovery-key` solo cuando la antigua clave de recuperación deba dejar de desbloquear la nueva copia de seguridad.

9. Si todavía no existe ninguna copia de seguridad de claves en el servidor, crea una para futuras recuperaciones:

   ```bash
   openclaw matrix verify bootstrap
   ```

## Mensajes habituales y su significado

`Failed migrating legacy Matrix client storage: ...`

- Significado: la alternativa del cliente de Matrix encontró un estado auxiliar basado en archivos, pero no se pudo importar a SQLite. OpenClaw revierte los movimientos completados y cancela esa alternativa, en lugar de iniciarse silenciosamente con un almacén nuevo.
- Qué hacer: revisa los permisos o conflictos del sistema de archivos, conserva intacto el estado anterior y vuelve a intentarlo después de corregir el error.

`Matrix is installed from a custom path: ...`

- Significado: Matrix está fijado a una instalación basada en una ruta, por lo que las actualizaciones de la rama principal no lo sustituyen automáticamente por el paquete predeterminado de Matrix.
- Qué hacer: vuelve a instalarlo con `openclaw plugins install @openclaw/matrix` cuando se quiera regresar al plugin predeterminado de Matrix.

`Matrix is installed from a custom path that no longer exists: ...`

- Significado: el registro de instalación del plugin apunta a una ruta local que ya no existe.
- Qué hacer: vuelve a instalarlo con `openclaw plugins install @openclaw/matrix` o, si se ejecuta desde una copia de trabajo del repositorio, con `openclaw plugins install ./path/to/local/matrix-plugin`. `openclaw doctor --fix` también puede eliminar las referencias obsoletas al plugin de Matrix.

### Mensajes de recuperación manual

`openclaw matrix verify status` y `openclaw matrix verify backup status` muestran una línea `Backup issue:` junto con indicaciones `Next steps:` cuando la copia de seguridad de claves de sala no está en buen estado en este dispositivo:

| Problema de la copia de seguridad                                     | Significado                                        | Solución                                                                                                                                  |
| --------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `no room-key backup exists on the homeserver`                         | no hay nada que restaurar                          | `openclaw matrix verify bootstrap` para crear una copia de seguridad de claves de sala                                                        |
| `backup decryption key is not loaded on this device`                  | la clave existe, pero no está activa aquí          | `openclaw matrix verify backup restore`; si aún no se puede cargar la clave, proporciona la clave de recuperación mediante `--recovery-key-stdin`     |
| `backup decryption key could not be loaded from secret storage (...)` | no se pudo cargar el almacenamiento secreto o no es compatible | proporciona la clave de recuperación: `printf '%s\n' "$MATRIX_RECOVERY_KEY" \| openclaw matrix verify backup restore --recovery-key-stdin`                                         |
| `backup key mismatch (...)`                                           | la clave almacenada no coincide con la copia de seguridad activa del servidor | vuelve a ejecutar `verify backup restore --recovery-key-stdin` con la clave de la copia de seguridad activa del servidor o `verify backup reset --yes` para establecer una nueva base de referencia |
| `backup signature chain is not trusted by this device`                | el dispositivo aún no confía en la cadena de firma cruzada | `verify device --recovery-key-stdin` y, después, `verify self` desde otro cliente verificado si la confianza sigue incompleta                 |
| `backup exists but is not active on this device`                      | existe una copia de seguridad en el servidor, pero la sesión local está inactiva | verifica primero el dispositivo y vuelve a comprobarlo con `openclaw matrix verify backup status`                                      |
| `backup trust state could not be fully determined`                    | el diagnóstico no fue concluyente                  | `openclaw matrix verify status --verbose`                                                                                                 |

Otros errores de recuperación:

`Matrix recovery key is required`

- Significado: se intentó realizar un paso de recuperación sin proporcionar una clave de recuperación cuando era necesaria.
- Qué hacer: vuelve a ejecutar el comando con `--recovery-key-stdin`, por ejemplo, `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin`.

`Invalid Matrix recovery key: ...`

- Significado: no se pudo analizar la clave proporcionada o esta no coincidía con el formato esperado.
- Qué hacer: vuelve a intentarlo con la clave de recuperación exacta obtenida del cliente de Matrix o de la exportación de claves de recuperación.

`Matrix recovery key was applied, but this device still lacks full Matrix identity trust.`

- Significado: la clave de recuperación desbloqueó material utilizable de la copia de seguridad, pero Matrix todavía no ha establecido una confianza completa en la identidad de firma cruzada para este dispositivo. Comprueba si la salida del comando contiene `Recovery key accepted`, `Backup usable`, `Cross-signing verified` y `Device verified by owner`.
- Qué hacer: ejecuta `openclaw matrix verify self`, acepta la solicitud en otro cliente de Matrix, compara la SAS y escribe `yes` solo cuando coincida. Utiliza `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify bootstrap --recovery-key-stdin --force-reset-cross-signing` únicamente cuando se quiera sustituir deliberadamente la identidad de firma cruzada actual.

Si se acepta la pérdida del historial cifrado antiguo irrecuperable, se puede restablecer la base de referencia actual de la copia de seguridad con `openclaw matrix verify backup reset --yes`. Cuando el secreto almacenado de la copia de seguridad está dañado, ese restablecimiento también repara el almacenamiento secreto para que la nueva clave de copia de seguridad pueda cargarse correctamente después del reinicio.

## Si el historial cifrado sigue sin recuperarse

Ejecuta estas comprobaciones en orden:

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin --verbose
```

Si la copia de seguridad se restaura correctamente, pero todavía falta el historial de algunas salas antiguas, es probable que el plugin anterior nunca guardara esas claves en una copia de seguridad.

## Si se desea empezar de cero para los mensajes futuros

Si se acepta la pérdida del historial cifrado antiguo irrecuperable y solo se desea establecer una base de referencia limpia para las copias de seguridad futuras, ejecuta estos comandos en orden:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Si el dispositivo sigue sin verificar después de esto, finaliza la verificación desde el cliente de Matrix comparando los emojis o códigos decimales de SAS y confirmando que coincidan.

## Temas relacionados

- [Matrix](/es/channels/matrix): configuración del canal.
- [Reglas push de Matrix](/es/channels/matrix-push-rules): enrutamiento de notificaciones.
- [Doctor](/es/gateway/doctor): comprobación del estado y activación automática de la migración.
- [Guía de migración](/es/install/migrating): todas las rutas de migración (traslados entre máquinas e importaciones entre sistemas).
- [Plugins](/es/tools/plugin): instalación y registro de plugins.
