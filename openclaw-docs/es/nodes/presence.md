---
read_when:
    - Quieres que OpenClaw identifique el Mac activo
    - Se está depurando la actividad de la última entrada o la selección del nodo activo
    - Se desea comprender el enrutamiento de las notificaciones de conexión de nodos
summary: Detecta el Mac que se utilizó más recientemente y dirige allí las alertas del Node
title: Presencia activa en el ordenador
x-i18n:
    generated_at: "2026-07-26T04:44:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f1d1d0e98b1f3b7478cf80696dc693677b57897b07260cce30938e9187c314
    source_path: nodes/presence.md
    workflow: 16
---

La presencia activa en el ordenador indica al Gateway qué Node de macOS conectado recibió
la entrada física de ratón o teclado más reciente. OpenClaw utiliza esa señal para
marcar un Mac como `active`, proporcionar al agente una indicación estable del Node activo y dirigir
las alertas de conexión de Nodes al ordenador donde es más probable que esté presente.

Esto es independiente de la [presencia del sistema](/es/concepts/presence), que es la lista en tiempo real
de clientes del Gateway, y de las balizas persistentes `node.presence.alive`, que
registran cuándo se activó por última vez un Node móvil sin considerarlo conectado.

## Requisitos

- La aplicación OpenClaw para macOS está emparejada y conectada en modo Node.
- **Settings -> Permissions -> Active computer detection** está habilitado. Está deshabilitado de forma predeterminada.
- Se ha concedido el permiso **Accessibility** a la aplicación OpenClaw firmada.
- Para las alertas de conexión, también se ha concedido el permiso **Notifications** y el
  Node Mac expone `system.notify`.

Actualmente, la generación de informes de actividad está implementada por el Node nativo de macOS. Los hosts de Nodes de iOS,
Android, watchOS y sin interfaz gráfica pueden informar del estado de conexión o de última presencia
en segundo plano, pero no compiten por la designación de ordenador activo.

## Comprobar el ordenador activo

1. En la aplicación para macOS, abra **Settings -> Permissions**, habilite
   **Active computer detection** y conceda **Accessibility** en la configuración del sistema de macOS.
2. Confirme que el Node Mac está conectado:

   ```bash
   openclaw nodes status --connected
   ```

3. Mueva el ratón o pulse una tecla en ese Mac y, a continuación, ejecute:

   ```bash
   openclaw nodes status
   openclaw nodes describe --node <node-id-or-name>
   ```

El Mac apto con actividad más reciente se marca como `active`. La salida de estado muestra la antigüedad de su última entrada;
`describe` expone `active`, `lastActiveAtMs` y `presenceUpdatedAtMs`.
La actividad se agrupa de forma intencionada, por lo que la pantalla puede tardar hasta unos 15
segundos en reflejar otra entrada después de un informe reciente.

## Cómo se convierte la actividad en presencia

El informador de macOS consulta el reloj de inactividad del sistema HID cada dos segundos. Informa
una vez cuando una conexión de Node queda lista y, después, comunica la actividad física más reciente
como máximo una vez cada 15 segundos. Mientras está inactivo, envía una señal de mantenimiento
cada tres minutos. La duración de inactividad se limita a 30 días para que una muestra muy antigua
no pueda avanzar con el tiempo y convertirse incorrectamente en la del ordenador más reciente.

Al deshabilitar **Active computer detection**, se detiene la consulta y se envía un evento autenticado
de eliminación a través de la conexión actual del Node. El Gateway elimina inmediatamente
las marcas de tiempo de actividad conservadas de ese Mac y vuelve a calcular el ordenador activo;
las demás capacidades del Node y el trabajo en curso permanecen conectados. Si el Gateway conectado
es anterior a esta acción de eliminación, el Node Mac vuelve a conectarse una vez para que la limpieza
por desconexión pueda eliminar en su lugar la actividad conservada.

El Gateway acepta la actividad solo cuando se cumplen todas estas condiciones:

- el evento pertenece a la conexión autenticada actual de ese identificador de Node;
- el Node tiene el permiso efectivo `accessibility: true`;
- la carga útil contiene un valor entero acotado `idleSeconds`.

El Gateway resta `idleSeconds` de su propio tiempo de observación para obtener
`lastActiveAtMs`. Nunca confía en una marca de tiempo del reloj del sistema proporcionada por un Node. Entre
los Mac conectados aptos, prevalece el valor `lastActiveAtMs` más reciente; en caso de empate, se utiliza la actualización de presencia
más reciente.

La presencia es local al proceso y está vinculada a la conexión. Desconectar la sesión
actual, sustituirla por otra sesión con el mismo identificador de Node o revocar
Accessibility elimina el estado de actividad de ese Node y vuelve a calcular el Mac activo.

## Privacidad y contexto del modelo

El uso compartido de la actividad está deshabilitado de forma predeterminada y es independiente de la concesión de Accessibility
utilizada para la automatización de la interfaz de usuario. OpenClaw envía la duración de inactividad, no el contenido de la entrada. No envía valores de teclas,
coordenadas del ratón, nombres de aplicaciones, títulos de ventanas ni eventos de entrada sin procesar. El
informador de macOS lee el estado del hardware HID, por lo que los eventos sintéticos de control
del ordenador no hacen que un Mac automatizado parezca ser el ordenador utilizado
físicamente.

La actividad continua no crea eventos del sistema visibles para el modelo. La línea dinámica
del entorno de ejecución solo contiene el identificador autenticado del Node:

```text
active_node=<node-id>
```

Las marcas de tiempo exactas y los nombres para mostrar controlados por los Nodes se mantienen fuera del prompt para
evitar la inyección de prompts y la renovación constante de la caché. Cuando el agente necesita detalles actuales,
la herramienta `nodes` puede leer `node.list` o `node.describe` en su lugar.

## Cómo se dirigen las alertas de conexión

Después de que un Node complete su primer protocolo de enlace correcto con el Gateway tras la aprobación,
OpenClaw espera 750 milisegundos para que el Mac que se está conectando pueda enviar su primera
muestra de actividad. Después, intenta utilizar el Mac conectado con capacidad de notificación que tenga la
actividad más reciente.

- Si la entrega principal se realiza correctamente, ningún otro Mac recibe la alerta.
- Si no hay ningún Mac activo disponible o la entrega principal falla, OpenClaw espera cinco
  segundos e intenta usar todos los demás Mac conectados que exponen `system.notify`.
- Las reconexiones posteriores son silenciosas. El Gateway registra la conexión correcta
  en los metadatos de emparejamiento, por lo que un reinicio del Gateway no vuelve a reproducir las alertas de todos los
  Nodes conectados anteriormente.

Las alertas están vinculadas a la identidad autenticada del Node. Una sesión de sustitución para
el mismo Node asume su alerta pendiente de primera conexión; si ese Node ya no está
conectado cuando se ejecuta la entrega, la alerta se cancela.

## Solución de problemas

| Síntoma                                    | Comprobación                                                                                                                                                                |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ninguna fila está marcada como `active` | Confirme que la detección del ordenador activo está habilitada, que hay un Node nativo de macOS conectado y que `openclaw nodes describe --node <id>` muestra `permissions.accessibility: true`.              |
| El Mac incorrecto sigue activo             | Utilice físicamente ese Mac, espere a que transcurra el intervalo de agrupación y vuelva a ejecutar `openclaw nodes status`. Las acciones sintéticas de control del ordenador no cuentan. |
| Desaparecen los datos de la última entrada | Compruebe si el Mac se desconectó, si se sustituyó su sesión de Node o si se revocó Accessibility. Cada condición elimina intencionadamente la actividad.                      |
| La alerta aparece en varios Mac            | La entrega principal no estaba disponible o falló, por lo que se ejecutó la alternativa retrasada. Verifique que el Mac activo esté conectado, permita notificaciones y exponga `system.notify`. |
| El agente no menciona el Mac activo        | Inicie un nuevo turno después de que cambie la actividad. La indicación del entorno de ejecución es estable y compacta; utilice la herramienta `nodes` para consultar los metadatos actuales exactos. |

Para recuperar los permisos de TCC, consulte [permisos de macOS](/es/platforms/mac/permissions). Para problemas de
conexión de Nodes y errores de comandos, consulte [Solución de problemas de Nodes](/es/nodes/troubleshooting).

## Contenido relacionado

- [Nodes](/es/nodes)
- [CLI de Nodes](/es/cli/nodes)
- [Presencia del sistema](/es/concepts/presence)
- [Protocolo del Gateway](/es/gateway/protocol#presence)
- [Aplicación para macOS](/es/platforms/macos)
