---
read_when:
    - Generación o revisión de planes de `openclaw secrets apply`
    - Depuración de errores de `Invalid plan target path`
    - Comprender el comportamiento de la validación del tipo y la ruta de destino
summary: 'Contrato para planes de `secrets apply`: validación de objetivos, coincidencia de rutas y ámbito objetivo de `auth-profiles.json`'
title: Contrato del plan de aplicación de secretos
x-i18n:
    generated_at: "2026-07-26T05:12:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ee8afd958646930af4db3bbad08e033ff79da48890a989d72b361abcbda3bb
    source_path: gateway/secrets-plan-contract.md
    workflow: 16
---

Esta página define el contrato estricto que aplica `openclaw secrets apply`. Si un destino no cumple estas reglas, la aplicación falla antes de modificar cualquier archivo.

## Requisitos del archivo de plan

`openclaw secrets apply --from <plan.json>` acepta archivos normales de hasta 16 MiB (16,777,216 bytes). El límite se aplica al archivo serializado completo, incluidos los espacios en blanco. Los directorios, las FIFO, los archivos de dispositivo y los archivos que superen el límite se rechazan antes del análisis de JSON o de la validación de destinos.

`openclaw secrets configure --plan-out <plan.json>` aplica el mismo límite a la salida serializada en UTF-8 antes de crear el archivo. Los planes escritos manualmente y los generadores de planes externos también deben mantener el archivo serializado dentro de este límite.

## Estructura del archivo de plan

`openclaw secrets apply --from <plan.json>` espera un arreglo `targets` de destinos del plan:

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

`openclaw secrets configure` genera planes con esta estructura. También se puede escribir o editar uno manualmente.

## Inserciones o actualizaciones y eliminaciones de proveedores

Los planes también pueden incluir dos campos opcionales de nivel superior que modifican el mapa `secrets.providers` junto con las escrituras de cada destino:

- `providerUpserts` -- un objeto cuyas claves son alias de proveedores. Cada valor es una definición de proveedor (la misma estructura aceptada en `secrets.providers.<alias>` dentro de `openclaw.json`, por ejemplo, un proveedor `exec` o `file`).
- `providerDeletes` -- un arreglo de alias de proveedores que se eliminarán.

`providerUpserts` se ejecuta antes de `targets`, por lo que un `target.ref.provider` puede hacer referencia a un alias de proveedor que el mismo plan introduce en `providerUpserts`. Sin este orden, los planes que hacen referencia a un alias que aún no está configurado en `openclaw.json` fallan con `provider "<alias>" is not configured`.

```json5
{
  version: 1,
  protocolVersion: 1,
  providerUpserts: {
    onepassword_anthropic: {
      source: "exec",
      command: "/usr/bin/op",
      args: ["read", "op://Vault/Anthropic/credential"],
    },
  },
  providerDeletes: ["legacy_unused_alias"],
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.anthropic.apiKey",
      pathSegments: ["models", "providers", "anthropic", "apiKey"],
      providerId: "anthropic",
      ref: { source: "exec", provider: "onepassword_anthropic", id: "credential" },
    },
  ],
}
```

Los proveedores de ejecución introducidos mediante `providerUpserts` siguen sujetos a las reglas de consentimiento para la ejecución descritas en [Comportamiento del consentimiento para proveedores de ejecución](#exec-provider-consent-behavior): los planes que contienen proveedores de ejecución requieren `--allow-exec` en modo de escritura.

## Ámbito de destinos admitidos

Se aceptan destinos de plan para las rutas de credenciales admitidas en [Superficie de credenciales SecretRef](/es/reference/secretref-credential-surface).

## Comportamiento de los tipos de destino

`target.type` debe ser un tipo de destino reconocido y el valor normalizado de `target.path` debe coincidir con la estructura de ruta registrada para ese tipo.

Algunos tipos de destino aceptan un alias de compatibilidad como `target.type` para los planes existentes, además de su nombre de tipo canónico:

| Tipo canónico                        | Alias aceptado                                  |
| ------------------------------------ | ----------------------------------------------- |
| `models.providers.apiKey`            | `models.providers.*.apiKey`                     |
| `skills.entries.apiKey`              | `skills.entries.*.apiKey`                       |
| `channels.googlechat.serviceAccount` | `channels.googlechat.accounts.*.serviceAccount` |

## Reglas de validación de rutas

Cada destino se valida con todas las reglas siguientes:

- `type` debe ser un tipo de destino reconocido.
- `path` debe ser una ruta de puntos no vacía.
- `pathSegments` puede omitirse. Si se proporciona, debe normalizarse exactamente a la misma ruta que `path`.
- Se rechazan los segmentos prohibidos: `__proto__`, `prototype`, `constructor`.
- La ruta normalizada debe coincidir con la estructura de ruta registrada para el tipo de destino.
- Si se establece `providerId` o `accountId`, debe coincidir con el identificador codificado en la ruta.
- Los destinos `auth-profiles.json` requieren `agentId`.
- Al crear una nueva asignación `auth-profiles.json`, incluya `authProfileProvider`.

## Comportamiento ante fallos

Si un destino no supera la validación, la aplicación finaliza con un error como el siguiente:

```text
Ruta de destino del plan no válida para models.providers.apiKey: models.providers.openai.baseUrl
```

No se confirma ninguna escritura para un plan no válido: la resolución de destinos y la validación de rutas se ejecutan antes de modificar cualquier archivo. Por separado, una vez que un plan válido comienza a escribir, la aplicación crea primero una instantánea de cada archivo afectado y restaura esas instantáneas si falla una escritura posterior de la misma ejecución, de modo que una escritura parcial nunca deje sin sincronizar la configuración, los perfiles de autenticación ni el estado del entorno.

## Comportamiento del consentimiento para proveedores de ejecución

- `--dry-run` omite de forma predeterminada las comprobaciones de SecretRef de ejecución.
- Los planes que contienen SecretRefs o proveedores de ejecución se rechazan en modo de escritura a menos que se establezca `--allow-exec`.
- Al validar o aplicar planes que contengan elementos de ejecución, pase `--allow-exec` tanto en los comandos de simulación como en los de escritura.

## Notas sobre el ámbito del entorno de ejecución y la auditoría

- Las entradas `auth-profiles.json` que solo contienen referencias (`keyRef`/`tokenRef`) se incluyen en la resolución de credenciales del entorno de ejecución y en la cobertura de auditoría.
- `secrets apply` escribe los destinos `openclaw.json` admitidos, los destinos `auth-profiles.json` admitidos y tres pasadas opcionales de limpieza, todas activadas de forma predeterminada: `scrubEnv` (elimina los valores de texto sin formato migrados de los archivos `.env` en los directorios de estado efectivo y de configuración activa), `scrubAuthProfilesForProviderTargets` (elimina los residuos de texto sin formato o de referencias sin usar en `auth-profiles.json` para los proveedores que un plan acaba de migrar) y `scrubLegacyAuthJson` (elimina las entradas `api_key` migradas de los almacenes `auth.json` heredados). Establezca cualquiera de `options.scrubEnv`, `options.scrubAuthProfilesForProviderTargets` o `options.scrubLegacyAuthJson` en `false` dentro del plan para omitir esa pasada.

## Comprobaciones del operador

```bash
# Validar el plan sin realizar escrituras
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# A continuación, aplicarlo realmente
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# Para los planes que contienen elementos de ejecución, dar el consentimiento explícitamente en ambos modos
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

Si la aplicación falla con un mensaje de ruta de destino no válida, vuelva a generar el plan con `openclaw secrets configure` o corrija la ruta de destino para que use una de las estructuras admitidas indicadas anteriormente.

## Documentación relacionada

- [Administración de secretos](/es/gateway/secrets)
- [CLI `secrets`](/es/cli/secrets)
- [Superficie de credenciales SecretRef](/es/reference/secretref-credential-surface)
- [Referencia de configuración](/es/gateway/configuration-reference)
