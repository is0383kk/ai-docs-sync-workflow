---
read_when:
    - Está aprobando solicitudes de emparejamiento de dispositivos
    - Es necesario rotar o revocar los tokens de dispositivo
summary: Referencia de la CLI para `openclaw devices` (emparejamiento de dispositivos + rotación/revocación de tokens)
title: Dispositivos
x-i18n:
    generated_at: "2026-07-26T05:02:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fb10f7a484fec06bfa5e53ae50181b12a9724746176bbace330ec468235494
    source_path: cli/devices.md
    workflow: 16
---

# `openclaw devices`

Gestiona las solicitudes de emparejamiento de dispositivos y los tokens con ámbito de dispositivo.

## Opciones comunes

- `--url <url>`: URL de WebSocket del Gateway (el valor predeterminado es `gateway.remote.url` cuando está configurada)
- `--token <token>`: token del Gateway (si es necesario)
- `--password <password>`: contraseña del Gateway (autenticación mediante contraseña)
- `--timeout <ms>`: tiempo de espera de RPC
- `--json`: salida JSON (recomendada para scripts)

<Warning>
Al establecer `--url`, la CLI no recurre a las credenciales de la configuración ni del entorno. Proporcione `--token` o `--password` explícitamente; de lo contrario, el comando genera un error.
</Warning>

## Comandos

### `openclaw devices list`

Enumera las solicitudes de emparejamiento pendientes y los dispositivos emparejados.

```bash
openclaw devices list
openclaw devices list --json
```

Para una solicitud pendiente en un dispositivo ya emparejado, la salida muestra el acceso solicitado junto al acceso aprobado actualmente para el dispositivo, de modo que las ampliaciones de ámbito o rol sean visibles en lugar de parecer un emparejamiento perdido.

Los nombres para mostrar de los dispositivos emparejados usan esta precedencia: etiqueta del operador (`operatorLabel` de `devices rename`), luego `displayName` del cliente, luego `clientId` y, por último, `deviceId`.

### `openclaw devices approve [requestId] [--latest]`

Aprueba una solicitud de emparejamiento pendiente mediante el `requestId` exacto. Omitir `requestId`, o proporcionar `--latest`, solo muestra una vista previa de la solicitud pendiente más reciente y finaliza (código 1); vuelva a ejecutar el comando con el ID de solicitud exacto para aprobarla.

```bash
openclaw devices approve
openclaw devices approve <requestId>
openclaw devices approve --latest
```

<Note>
Si un dispositivo vuelve a intentar el emparejamiento con datos de autenticación distintos (rol, ámbitos o clave pública), OpenClaw sustituye la entrada pendiente anterior por un nuevo `requestId`. Ejecute `openclaw devices list` justo antes de la aprobación para obtener el ID actual.
</Note>

Comportamiento de la aprobación:

- Si el dispositivo ya está emparejado y solicita ámbitos más amplios u otro rol, OpenClaw conserva la aprobación existente y crea una nueva solicitud de ampliación pendiente. Compare `Requested` con `Approved` en `openclaw devices list`, o muestre una vista previa con `--latest`, antes de aprobar.
- La aprobación de un rol `node` u otro rol que no sea de operador requiere `operator.admin`. `operator.pairing` es suficiente para aprobar dispositivos de operador, pero solo cuando los ámbitos de operador solicitados se mantienen dentro de los ámbitos propios del solicitante. Consulte [Ámbitos de operador](/es/gateway/operator-scopes).
- Si `gateway.nodes.pairing.autoApproveCidrs` está configurado, las primeras solicitudes `role: node` procedentes de direcciones IP de cliente coincidentes pueden aprobarse automáticamente antes de aparecer en esta lista. Está deshabilitado de forma predeterminada y nunca se aplica a clientes de operador/navegador ni a solicitudes de ampliación.
- `gateway.nodes.pairing.sshVerify` (activado de forma predeterminada) aprueba automáticamente las primeras solicitudes `role: node` cuando el Gateway verifica mediante SSH la clave del dispositivo con el host del Node. Por lo tanto, las solicitudes pueden pasar al estado aprobado poco después de aparecer. Establezca `sshVerify: false` para deshabilitar la verificación SSH; esto es independiente de `autoApproveCidrs`, por lo que también debe desactivar este último para que el emparejamiento sea exclusivamente manual.

### `openclaw devices reject <requestId>`

Rechaza una solicitud pendiente de emparejamiento de dispositivo.

```bash
openclaw devices reject <requestId>
```

### `openclaw devices remove <deviceId>`

Elimina una entrada de dispositivo emparejado.

```bash
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

Un solicitante autenticado con un token de dispositivo emparejado solo puede eliminar la entrada de su **propio** dispositivo. Para eliminar otro dispositivo se requiere `operator.admin`.

### `openclaw devices rename --device <id> --name <label>`

Asigna una etiqueta de operador a un dispositivo emparejado. Las etiquetas son estado del propietario: se conservan tras reparaciones del emparejamiento y nuevas aprobaciones de roles, y no cambian el `deviceId` estable.

```bash
openclaw devices rename --device <deviceId> --name "Kitchen Mac"
openclaw devices rename --device <deviceId> --name "Kitchen Mac" --json
```

- `--name` es obligatorio, se recorta, no puede estar vacío y tiene un límite de 64 caracteres.
- Las superficies de visualización (lista de la CLI e inventario de la interfaz de control) dan preferencia a la etiqueta del operador frente al nombre para mostrar comunicado por el cliente.
- Un solicitante de dispositivo emparejado que no sea administrador solo puede cambiar el nombre de su **propio** dispositivo. Para cambiar el nombre de otro dispositivo se requiere `operator.admin`.

### `openclaw devices clear --yes [--pending]`

Borra dispositivos emparejados de forma masiva. Requiere `--yes`.

```bash
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

`--pending` también rechaza todas las solicitudes de emparejamiento pendientes.

### `openclaw devices rotate --device <id> --role <role> [--scope <scope...>]`

Rota un token de dispositivo para un rol y, opcionalmente, actualiza sus ámbitos.

```bash
openclaw devices rotate --device <deviceId> --role operator --scope operator.read --scope operator.write
```

- El rol de destino ya debe existir en el contrato de emparejamiento aprobado de ese dispositivo; la rotación no puede crear un nuevo rol no aprobado.
- Si se omite `--scope`, se reutilizan los ámbitos aprobados almacenados en caché del token guardado en conexiones posteriores. Al proporcionar valores `--scope` explícitos, se reemplaza el conjunto de ámbitos almacenado para futuras reconexiones con tokens en caché.
- Un solicitante de dispositivo emparejado que no sea administrador solo puede rotar el token de su **propio** dispositivo, y el conjunto de ámbitos de destino debe mantenerse dentro de los ámbitos de operador propios del solicitante; la rotación no puede crear ni conservar un token más amplio que el que ya posee el solicitante.

Devuelve los metadatos de rotación como JSON. Si el solicitante rota su propio token mientras está autenticado con ese token de dispositivo, la respuesta incluye el token de reemplazo para que el cliente pueda conservarlo antes de volver a conectarse. Las rotaciones compartidas o de administrador nunca muestran el token de portador.

### `openclaw devices revoke --device <id> --role <role>`

Revoca un token de dispositivo para un rol.

```bash
openclaw devices revoke --device <deviceId> --role node
```

Un solicitante de dispositivo emparejado que no sea administrador solo puede revocar el token de su **propio** dispositivo. Para revocar el token de otro dispositivo se requiere `operator.admin`. El conjunto de ámbitos de destino también debe estar dentro de los ámbitos de operador propios del solicitante; los solicitantes que solo tengan permisos de emparejamiento no pueden revocar tokens de operador con permisos de administración/escritura.

## Notas

- Estos comandos requieren el ámbito `operator.pairing` (o `operator.admin`). Los roles de dispositivo que no sean de operador siempre requieren `operator.admin`; consulte [Ámbitos de operador](/es/gateway/operator-scopes).
- La rotación y revocación de tokens se mantienen dentro del conjunto de roles de emparejamiento aprobado y de la referencia de ámbitos del dispositivo. Una entrada de token en caché aislada no concede un destino de administración de tokens.
- En las sesiones con tokens de dispositivos emparejados, la administración entre dispositivos (`remove`, `rename`, `rotate`, `revoke`) se limita al dispositivo propio, salvo que el solicitante tenga `operator.admin`.
- La rotación de tokens devuelve un nuevo token (confidencial); trátelo como un secreto.
- Si el ámbito de emparejamiento no está disponible en la interfaz de bucle invertido local y no se proporciona ningún `--url` explícito, `list`/`approve` pueden recurrir al estado de emparejamiento local.

## Lista de comprobación para recuperar la sincronización de tokens

Use esta lista cuando la interfaz de control u otros clientes sigan fallando con `AUTH_TOKEN_MISMATCH`, `AUTH_DEVICE_TOKEN_MISMATCH` o `AUTH_SCOPE_MISMATCH`.

1. Confirme el origen actual del token del Gateway:

   ```bash
   openclaw config get gateway.auth.token
   ```

2. Enumere los dispositivos emparejados e identifique el ID del dispositivo afectado:

   ```bash
   openclaw devices list
   ```

3. Rote el token de operador del dispositivo afectado:

   ```bash
   openclaw devices rotate --device <deviceId> --role operator
   ```

4. Si la rotación no es suficiente, elimine el emparejamiento obsoleto y vuelva a aprobarlo:

   ```bash
   openclaw devices remove <deviceId>
   openclaw devices list
   openclaw devices approve <requestId>
   ```

5. Vuelva a intentar la conexión del cliente con el token o la contraseña compartidos actuales.

Notas:

- Precedencia normal de autenticación al reconectar: primero el token o la contraseña compartidos explícitos, luego `deviceToken` explícito, después el token de dispositivo almacenado y, por último, el token de arranque.
- La recuperación de confianza de `AUTH_TOKEN_MISMATCH` puede enviar temporalmente tanto el token compartido como el token de dispositivo almacenado en un único reintento limitado.
- `AUTH_SCOPE_MISMATCH` significa que se reconoció el token de dispositivo, pero no incluye el conjunto de ámbitos solicitado; corrija el contrato de aprobación del emparejamiento o de los ámbitos antes de cambiar la autenticación compartida del Gateway.

Relacionado:

- [Solución de problemas de autenticación del panel](/es/web/dashboard#if-you-see-unauthorized-1008)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting#dashboard-control-ui-connectivity)

## Aprobación de la primera ejecución de Paperclip / `openclaw_gateway`

Los agentes de Paperclip que se conectan mediante el adaptador `openclaw_gateway` pasan por la misma aprobación de emparejamiento de dispositivos en la primera ejecución que cualquier otro cliente nuevo. Si Paperclip informa de `openclaw_gateway_pairing_required`, apruebe el dispositivo pendiente y vuelva a intentarlo.

```bash
openclaw devices approve --latest
```

La vista previa muestra el comando `openclaw devices approve <requestId>` exacto; verifique los detalles y, a continuación, vuelva a ejecutar ese comando con el ID de solicitud para aprobarlo. Para un Gateway remoto o credenciales explícitas, proporcione las mismas opciones tanto al mostrar la vista previa como al aprobar:

```bash
openclaw devices approve --latest --url <gateway-ws-url> --token <gateway-token>
```

Para evitar tener que volver a aprobar después de cada reinicio, configure un `adapterConfig.devicePrivateKeyPem` persistente en Paperclip en lugar de permitir que genere una nueva identidad de dispositivo efímera en cada ejecución:

```json
{
  "adapterConfig": {
    "devicePrivateKeyPem": "<ed25519-private-key-pkcs8-pem>"
  }
}
```

Si la aprobación sigue fallando, ejecute primero `openclaw devices list` para confirmar que existe una solicitud pendiente.

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Nodes](/es/nodes)
