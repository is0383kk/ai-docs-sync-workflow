---
read_when:
    - Volver a resolver las referencias de secretos en tiempo de ejecución
    - Auditoría de residuos de texto sin formato y referencias sin resolver
    - Configuración de SecretRefs y aplicación de cambios de depuración unidireccional
summary: Referencia de la CLI para `openclaw secrets` (recargar, auditar, configurar, aplicar)
title: Secretos
x-i18n:
    generated_at: "2026-07-26T05:35:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

Gestiona las SecretRefs y mantiene en buen estado la instantánea activa del entorno de ejecución.

| Comando     | Función                                                                                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | RPC del Gateway (`secrets.reload`): vuelve a resolver las referencias y publica atómicamente la instantánea del entorno de ejecución que tiene en cuenta al propietario (sin escribir en la configuración); los fallos de propietarios aptos pueden publicarse como advertencias frías u obsoletas |
| `audit`     | Análisis de solo lectura de los almacenes de configuración, autenticación y modelos generados, así como de los residuos heredados, para detectar texto sin formato, referencias sin resolver y desviaciones de precedencia (las referencias de ejecución se omiten salvo que se use `--allow-exec`)                      |
| `configure` | Planificador interactivo para configurar proveedores, asignar destinos y realizar la comprobación previa (requiere una TTY)                                                                                                       |
| `apply`     | Ejecuta un plan guardado (`--dry-run` solo valida y omite de forma predeterminada las comprobaciones de ejecución; el modo de escritura rechaza los planes que contienen operaciones de ejecución salvo que se use `--allow-exec`) y después elimina los residuos de texto sin formato seleccionados |

Bucle recomendado para operadores:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

Si el plan incluye SecretRefs/proveedores `exec`, pasa `--allow-exec` tanto al comando `apply` de simulación como al de escritura.

Códigos de salida para la Pipeline de CI/puertas de control:

- `audit --check` devuelve `1` cuando se detectan hallazgos.
- Las referencias sin resolver devuelven `2` (independientemente de `--check`).

Relacionado: [Gestión de secretos](/es/gateway/secrets) · [Superficie de credenciales SecretRef](/es/reference/secretref-credential-surface) · [Seguridad](/es/gateway/security)

## Recargar la instantánea del entorno de ejecución

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Utiliza el método RPC del Gateway `secrets.reload`. Los propietarios en buen estado se actualizan de forma independiente. Los propietarios aptos con fallos solo pasan a estar obsoletos cuando sus identidades de referencia, definiciones de proveedor y contrato no secreto completo del propietario permanecen sin cambios; los fallos nuevos o modificados pasan a estar fríos. Esta activación degradada se completa correctamente e informa de `warningCount`. Los fallos estrictos o sin asignar devuelven un error y conservan la instantánea que estaba activa anteriormente.

Opciones: `--url <url>`, `--token <token>`, `--timeout <ms>`, `--json`.

## Auditoría

Analiza el estado de OpenClaw para detectar:

- almacenamiento de secretos en texto sin formato
- referencias sin resolver
- desviación de precedencia (credenciales `auth-profiles.json` que ocultan referencias `openclaw.json`)
- residuos `agents/*/agent/models.json` generados (valores `apiKey` del proveedor y encabezados confidenciales del proveedor)
- residuos heredados (entradas heredadas del almacén de autenticación, recordatorios de OAuth)

El análisis `.env` abarca el directorio de estado efectivo y el directorio que contiene la configuración activa. Cuando ambas rutas designan el mismo archivo, este se analiza una sola vez.

La detección de encabezados confidenciales del proveedor se basa en una heurística de nombres: marca los encabezados cuyo nombre coincide con fragmentos habituales de autenticación o credenciales (`authorization`, `x-api-key`, `token`, `secret`, `password`, `credential`).

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

Estructura del informe:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`, `skippedExecRefs`, `resolvabilityComplete`
- `summary`: `plaintextCount`, `unresolvedRefCount`, `shadowedRefCount`, `legacyResidueCount`
- códigos de hallazgo: `PLAINTEXT_FOUND`, `REF_UNRESOLVED`, `REF_SHADOWED`, `LEGACY_RESIDUE`

## Configurar (asistente interactivo)

Crea de forma interactiva cambios de proveedores y SecretRef, ejecuta la comprobación previa y, opcionalmente, los aplica:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

Flujo: primero se configuran los proveedores (añadir, editar o eliminar alias `secrets.providers`), después se asignan las credenciales (seleccionar campos y asignar referencias `{source, provider, id}`) y, por último, se realiza la comprobación previa y la aplicación opcional.

Opciones:

- `--providers-only`: configura solo `secrets.providers` y omite la asignación de credenciales
- `--skip-provider-setup`: omite la configuración de proveedores y asigna las credenciales a proveedores existentes
- `--agent <id>`: limita la detección de destinos y las escrituras de `auth-profiles.json` a un único almacén de agente
- `--allow-exec`: permite las comprobaciones de SecretRef de ejecución durante la comprobación previa/aplicación (puede ejecutar comandos del proveedor)

`--providers-only` y `--skip-provider-setup` no se pueden combinar.

Notas:

- Requiere una TTY interactiva.
- Selecciona los campos que contienen secretos en `openclaw.json`, además de `auth-profiles.json` para el ámbito del agente seleccionado; superficie canónica admitida: [Superficie de credenciales SecretRef](/es/reference/secretref-credential-surface).
- Permite crear nuevas asignaciones `auth-profiles.json` directamente en el flujo del selector.
- Ejecuta la resolución de comprobación previa antes de aplicar.
- Los planes generados activan de forma predeterminada las opciones de eliminación de datos (`scrubEnv`, `scrubAuthProfilesForProviderTargets`, `scrubLegacyAuthJson`). La aplicación es irreversible para los valores en texto sin formato eliminados.
- `--plan-out` se niega a crear un plan cuya forma serializada en UTF-8 supere 16 MiB (16,777,216 bytes), de acuerdo con el límite de entrada de `apply --from`.
- Sin `--apply`, la CLI sigue solicitando `Apply this plan now?` después de la comprobación previa.
- Con `--apply` (y sin `--yes`), la CLI solicita una confirmación adicional de la migración irreversible.
- `--json` muestra el plan y el informe de comprobación previa, pero sigue requiriendo una TTY interactiva.

### Seguridad del proveedor de ejecución

Las instalaciones de Homebrew suelen exponer binarios mediante enlaces simbólicos en `/opt/homebrew/bin/*`. Configura `allowSymlinkCommand: true` solo cuando sea necesario para rutas de gestores de paquetes de confianza, junto con `trustedDirs` (por ejemplo, `["/opt/homebrew"]`). En Windows, si no se puede verificar la ACL de una ruta del proveedor, OpenClaw adopta una política de denegación predeterminada; solo para rutas de confianza, configura `allowInsecurePath: true` en ese proveedor para omitir la comprobación de seguridad de la ruta.

## Aplicar un plan guardado

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` valida la comprobación previa sin escribir archivos; las comprobaciones de SecretRef de ejecución se omiten de forma predeterminada en la simulación. El modo de escritura rechaza los planes que contienen SecretRefs/proveedores de ejecución salvo que se use `--allow-exec`. Usa `--allow-exec` para habilitar las comprobaciones o la ejecución del proveedor de ejecución en cualquiera de los modos.

`--from` debe apuntar a un archivo normal de no más de 16 MiB (16,777,216 bytes). El límite de bytes se aplica al archivo serializado completo, incluidos los espacios en blanco.

Elementos que `apply` puede actualizar:

- `openclaw.json` (destinos SecretRef y altas/actualizaciones/eliminaciones de proveedores)
- `auth-profiles.json` (eliminación de datos de destinos de proveedores)
- residuos heredados de `auth.json`
- archivos `.env` de los directorios de estado efectivo y configuración activa, para claves secretas conocidas cuyos valores se migraron

Detalles del contrato del plan (rutas de destino permitidas, reglas de validación y semántica de los fallos): [Contrato del plan de aplicación de secretos](/es/gateway/secrets-plan-contract).

### Por qué no hay copias de seguridad para revertir

`secrets apply` no escribe intencionadamente copias de seguridad para revertir que contengan los valores antiguos en texto sin formato. La seguridad se basa en una comprobación previa estricta y una aplicación prácticamente atómica, con una restauración en memoria con el máximo esfuerzo en caso de fallo.

## Ejemplo

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

Si `audit --check` sigue informando de hallazgos de texto sin formato, actualiza las rutas de destino restantes indicadas y vuelve a ejecutar la auditoría.

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Gestión de secretos](/es/gateway/secrets)
- [SecretRefs de Vault](/es/plugins/vault)
