---
read_when:
    - Configurar o depurar el control remoto de Mac
summary: Flujo de la aplicación para macOS para controlar un Gateway remoto de OpenClaw
title: Control remoto
x-i18n:
    generated_at: "2026-07-26T05:46:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e558c39fa173a77bf11270a8961c14c6e2350dfc4f458da3633532513b98bf6
    source_path: platforms/mac/remote.md
    workflow: 16
---

Este flujo permite que la aplicación de macOS actúe como control remoto completo de un Gateway de OpenClaw que se ejecuta en otro host (equipo de escritorio/servidor). La aplicación se conecta directamente a URLs de Gateway de LAN/Tailnet de confianza o administra un túnel SSH cuando el Gateway remoto solo está disponible en loopback. Las comprobaciones de estado, el reenvío de activación por voz y el chat web reutilizan la misma configuración remota de _Settings -> General_.

## Modos

- **Local (este Mac)**: todo se ejecuta en el portátil; no interviene SSH.
- **Remoto mediante SSH (predeterminado)**: los comandos de OpenClaw se ejecutan en el host remoto. La aplicación abre una conexión SSH con `-o BatchMode`, la identidad/clave elegida y un reenvío de puerto local.
- **Remoto directo (ws/wss)**: sin túnel SSH; la aplicación se conecta directamente a la URL del Gateway (LAN, Tailscale, Tailscale Serve o un proxy inverso HTTPS público).

## Transportes remotos

- **Túnel SSH** (predeterminado): utiliza `ssh -N -L ...` para reenviar el puerto del Gateway a localhost. El Gateway ve la IP del Node como `127.0.0.1` porque el túnel usa loopback.
- **Directo (ws/wss)**: se conecta directamente a la URL del Gateway. El Gateway ve la IP real del cliente.

La aplicación desactiva la multiplexación de conexiones SSH y la ejecución en segundo plano posterior a la autenticación para sus propios procesos SSH, de modo que pueda supervisar y reiniciar el proceso exacto, aunque el alias seleccionado habilite `ControlMaster` o `ForkAfterAuthentication`.

La verificación de la clave de host SSH es estricta de forma predeterminada porque las credenciales del Gateway viajan por este túnel. Para utilizar el comportamiento de confianza propio de un alias SSH administrado, configure `--ssh-host-key-policy openssh` mediante `openclaw-mac configure-remote` o establezca directamente `gateway.remote.sshHostKeyPolicy` en `"openssh"`. Revise el alias y cualquier configuración del sistema o `Host *` que coincida antes de habilitar esta opción. Al cambiar el destino SSH (en la aplicación o mediante `configure-remote`), la política vuelve a `strict`, a menos que vuelva a habilitar explícitamente la opción para el nuevo destino.

En el modo de túnel SSH, los nombres de host de LAN/tailnet detectados se guardan como `gateway.remote.sshTarget`. La aplicación mantiene `gateway.remote.url` en el punto de conexión local del túnel (por ejemplo, `ws://127.0.0.1:18789`) para que la CLI, el chat web y el servicio de host del Node local utilicen el mismo transporte de loopback. Cuando la detección devuelve tanto direcciones IP de Tailnet sin procesar como nombres de host estables, la aplicación prefiere los nombres de Tailscale MagicDNS o LAN para que las conexiones resistan mejor los cambios de dirección. Si el puerto local del túnel difiere del puerto remoto del Gateway, establezca `gateway.remote.remotePort` en el puerto del host remoto.

En el modo remoto, la automatización del navegador pertenece al host de Node de la CLI, no al Node de la aplicación nativa de macOS. La aplicación inicia el servicio de host de Node instalado cuando es posible; para habilitar el control del navegador desde ese Mac, instálelo/inícielo con `openclaw node install ...` y `openclaw node start` (o ejecute `openclaw node run ...` en primer plano) y, a continuación, seleccione como destino ese Node con capacidad de navegador.

## Requisitos previos en el host remoto

1. Instale Node + pnpm y compile/instale la CLI de OpenClaw (`pnpm install && pnpm build && pnpm link --global`).
2. Asegúrese de que `openclaw` esté en PATH para los shells no interactivos (cree un enlace simbólico en `/usr/local/bin` o `/opt/homebrew/bin` si es necesario).
3. Para el transporte SSH: configure la autenticación SSH basada en claves. Se recomiendan las direcciones IP de Tailscale para mantener un acceso estable fuera de la LAN.

## Configuración de la aplicación de macOS

Para preconfigurar la aplicación sin el flujo de bienvenida, mediante SSH:

```bash
openclaw-mac configure-remote \
  --ssh-target user@gateway-host \
  --local-port 18789 \
  --remote-port 18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

O bien, para un Gateway que ya sea accesible desde una LAN o Tailnet de confianza, omita SSH por completo:

```bash
openclaw-mac configure-remote \
  --direct-url ws://192.168.0.202:18789 \
  --token "$OPENCLAW_GATEWAY_TOKEN"
```

`openclaw-mac connect`, `wizard` y `configure-remote` resuelven la configuración activa en este orden: `OPENCLAW_CONFIG_PATH`, después `$OPENCLAW_STATE_DIR/openclaw.json` y, por último, `~/.openclaw/openclaw.json`. Ambas formas de configuración escriben ese archivo activo, marcan la incorporación como completada y permiten que la aplicación controle el transporte seleccionado en el siguiente inicio. `--local-port`/`--remote-port` utilizan `18789` de forma predeterminada. Otras opciones: `--password`, `--identity <path>`, `--ssh-host-key-policy <strict|openssh>`, `--project-root <path>`, `--cli-path <path>`, `--json`. Ejecute `openclaw-mac configure-remote --help` para consultar la referencia completa.

Para realizar la configuración desde la interfaz de usuario:

1. Abra _Settings -> General_.
2. En **OpenClaw runs**, seleccione **Remote** y configure:
   - **Transport**: **SSH tunnel** o **Direct (ws/wss)**.
   - **SSH target**: `user@host` (`:port` opcional). Si el Gateway está en la misma LAN y se anuncia mediante Bonjour, selecciónelo en la lista de dispositivos detectados para rellenar automáticamente este campo.
   - **Gateway URL** (solo Direct): `wss://gateway.example.ts.net` (o `ws://...` para local/LAN).
   - **Identity file** (avanzado): ruta a la clave.
   - **Project root** (avanzado): ruta remota del checkout utilizada para los comandos.
   - **CLI path** (avanzado): ruta opcional a un punto de entrada/binario ejecutable de `openclaw` (se rellena automáticamente cuando se anuncia).
3. Pulse **Test remote**. Si se completa correctamente, significa que el `openclaw status --json` remoto se ejecutó de forma adecuada. Los fallos suelen indicar problemas con PATH o la CLI; el código de salida 127 significa que no se encontró la CLI en el sistema remoto.
4. Las comprobaciones de estado y el chat web ahora utilizan automáticamente el transporte seleccionado.

## Chat web

- **Túnel SSH**: se conecta al Gateway mediante el puerto de control WebSocket reenviado (18789 de forma predeterminada).
- **Directo (ws/wss)**: se conecta directamente a la URL configurada del Gateway.
- No existe un servidor HTTP independiente para el chat web.

## Permisos

- El host remoto necesita las mismas autorizaciones de TCC que el host local (Automatización, Accesibilidad, Grabación de pantalla, Micrófono, Reconocimiento de voz y Notificaciones). Ejecute una vez la incorporación en ese equipo para concederlas.
- Los Nodes anuncian el estado de sus permisos mediante `node.list` / `node.describe` para que los agentes sepan qué está disponible.

## Notas de seguridad

- Es preferible enlazar el host remoto a loopback y conectarse mediante SSH, Tailscale Serve o una URL directa de Tailnet/LAN de confianza.
- De forma predeterminada, el túnel SSH requiere una clave de host que ya sea de confianza. Confíe primero en la clave de host (añádala al archivo de hosts conocidos configurado) o establezca explícitamente `gateway.remote.sshHostKeyPolicy: "openssh"` para un alias administrado cuya política de confianza de OpenSSH acepte.
- Si enlaza el Gateway a una interfaz que no sea loopback, exija una autenticación válida del Gateway: token, contraseña o un proxy inverso con reconocimiento de identidad y `gateway.auth.mode: "trusted-proxy"`.
- Las conexiones directas `wss://` aplican una única política de certificados tanto al tráfico de operación/control como al Node complementario del Mac. Establezca `gateway.remote.tlsFingerprint` para fijar explícitamente el certificado. Si no se configura, la aplicación registra una fijación en el primer uso únicamente después de que la confianza normal de macOS se valide correctamente.
- Consulte [Seguridad](/es/gateway/security) y [Tailscale](/es/gateway/tailscale).

## Flujo de inicio de sesión de WhatsApp (remoto)

- Ejecute `openclaw channels login --channel whatsapp --verbose` **en el host remoto**. Escanee el código QR con WhatsApp en el teléfono.
- Vuelva a iniciar sesión en ese host si la autenticación caduca. La comprobación de estado muestra los problemas de vinculación.

## Solución de problemas

| Síntoma                                          | Causa / solución                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exit 127` / no encontrado                           | `openclaw` no está en PATH para shells que no son de inicio de sesión. Añádalo a `/etc/paths`, al archivo rc del shell o cree un enlace simbólico en `/usr/local/bin`/`/opt/homebrew/bin`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Falló la comprobación de estado                              | Compruebe la conectividad SSH, PATH y que se haya iniciado sesión en Baileys (WhatsApp) (`openclaw status --json`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| El chat web está bloqueado                                   | Confirme que el gateway se esté ejecutando en el host remoto y que el puerto reenviado coincida con el puerto WS del gateway; la interfaz de usuario requiere una conexión WS en buen estado.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| La IP del Node muestra `127.0.0.1`                        | Es lo esperado con el túnel SSH. Cambie **Transporte** a **Directo (ws/wss)** si desea que el gateway vea la IP real del cliente.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| El panel funciona, pero las capacidades del Mac están sin conexión | La conexión del operador/control está en buen estado, pero la conexión del Node complementario no está conectada o no dispone de su superficie de comandos. Abra la sección de dispositivos de la barra de menús y compruebe si el Mac está `paired · disconnected`. Las conexiones directas de operador y Node mediante `wss://` utilizan la misma política de certificados configurada o almacenada. En los endpoints de Tailscale Serve de confianza mediante `wss://*.ts.net`, los pines de certificados hoja almacenados y obsoletos se sustituyen tras la rotación del certificado y se vuelve a intentar automáticamente. Los pines configurados nunca rotan automáticamente; actualice `gateway.remote.tlsFingerprint` después de revisar el certificado nuevo o cambie a **Remoto mediante SSH**. |
| Activación por voz                                       | Las frases de activación se reenvían automáticamente en modo remoto; no se necesita ningún reenviador independiente.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

## Sonidos de notificación

Seleccione sonidos por notificación desde scripts con `openclaw nodes notify`, por ejemplo:

```bash
openclaw nodes notify --node <id> --title "Ping" --body "Gateway remoto listo" --sound Glass
```

No hay ningún conmutador global de sonido predeterminado en la aplicación; los invocadores eligen un sonido (o ninguno) para cada solicitud.

## Relacionado

- [Aplicación para macOS](/es/platforms/macos)
- [Acceso remoto](/es/gateway/remote)
