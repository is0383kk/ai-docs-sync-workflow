---
read_when:
    - Aloja varios dominios de confianza de inquilinos en una sola máquina
    - Necesita crear, inspeccionar, actualizar o eliminar celdas de la flota
summary: Referencia de la CLI para aprovisionar y gestionar celdas aisladas de OpenClaw por inquilino
title: Flota
x-i18n:
    generated_at: "2026-07-26T04:33:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: be589500e4715541f175caf0d5135a96baee4874e64c60c8b6f188ff1f70bc9f
    source_path: cli/fleet.md
    workflow: 16
---

# `openclaw fleet`

`openclaw fleet` administra instancias completas de OpenClaw denominadas **celdas**. Cada celda tiene su propio Gateway, estado, credenciales, cuentas de canales, contenedor y puerto de host exclusivo de loopback. Use una celda para cada límite de confianza entre inquilinos; no use un Gateway compartido como límite multiinquilino frente a entidades hostiles.

Fleet es **experimental**. Los nombres de los comandos, las opciones, los formatos de salida y el perfil del contenedor pueden cambiar entre versiones sin un período de obsolescencia.

Fleet admite Docker y Podman. La imagen predeterminada es `ghcr.io/openclaw/openclaw:latest`.

Fleet se prueba en hosts Linux y macOS. Actualmente, no se ha probado en hosts Windows.

## Inicio rápido

```bash
openclaw fleet create acme
openclaw fleet status acme
openclaw fleet list
```

`fleet create` muestra una sola vez el token de Gateway generado junto con la URL de la celda. Guarde el token de inmediato y, a continuación, configure las cuentas de canales de cada inquilino dentro de la celda de ese inquilino.

## Identificadores de inquilino

Los identificadores de inquilino deben coincidir con:

```text
^[a-z0-9](?:[a-z0-9-]{0,38}[a-z0-9])?$
```

Esto permite entre 1 y 40 letras minúsculas, dígitos y guiones internos. Un identificador debe comenzar y terminar con una letra o un dígito. Se rechazan las letras mayúsculas, los guiones bajos, las barras, los puntos, los espacios en blanco y las cadenas de recorrido como `../acme`.

El identificador pasa a formar parte del nombre del contenedor: `openclaw-cell-<tenant>`.

## `fleet create`

Cree una celda e iníciela:

```bash
openclaw fleet create acme
```

Cree una celda de Podman en un puerto fijo sin iniciarla:

```bash
openclaw fleet create acme \
  --runtime podman \
  --port 19125 \
  --no-start
```

Pase variables de entorno específicas del inquilino repitiendo `--env`:

```bash
openclaw fleet create acme \
  --env TZ=America/Los_Angeles \
  --env OPENCLAW_DISABLE_BONJOUR=1
```

Las claves de entorno usan letras, dígitos y guiones bajos, y no pueden comenzar con un dígito. Los valores deben ocupar una sola línea porque Fleet los pasa mediante un archivo de entorno protegido del entorno de ejecución. Fleet rechaza los intentos de sobrescribir las variables administradas de rutas del contenedor y del token de Gateway enumeradas en [Almacenamiento y disposición del contenedor](#storage-and-container-layout).

### Opciones de creación

| Opción                    | Valor predeterminado                               | Descripción                                                                                    |
| ------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `--image <ref>`           | `ghcr.io/openclaw/openclaw:latest`    | Imagen del contenedor para la celda.                                                                  |
| `--runtime <runtime>`     | `docker`                              | CLI del contenedor: `docker` o `podman`.                                                           |
| `--port <number>`         | Asignado automáticamente desde `19100`  | Puerto de host de loopback. Un puerto seleccionado explícitamente no debe pertenecer a otra celda registrada.    |
| `--memory <value>`        | `2g`                                  | Límite de memoria del contenedor en la sintaxis de Docker/Podman.                                                |
| `--cpus <value>`          | `2`                                   | Límite de CPU del contenedor.                                                                           |
| `--disk <size>`           | Ninguno                                  | Limita la capa escribible del contenedor cuando el backend de almacenamiento admite cuotas.                     |
| `--network <mode>`        | `bridge`                              | Modo de red saliente: `bridge` o `internal`.                                                 |
| `--pids-limit <number>`   | `512`                                 | Número máximo de procesos en el contenedor.                                                  |
| `--env <KEY=VALUE>`       | Ninguno                                  | Pasa una variable de entorno a la celda. Repita la opción para varios valores.                          |
| `--gateway-token <value>` | Token hexadecimal aleatorio de 32 caracteres | Usa un token de Gateway proporcionado en lugar de generar uno. Consulte [Administración de tokens](#token-handling). |
| `--no-start`              | La celda se inicia                           | Crea el contenedor sin iniciarlo.                                                      |
| `--json`                  | Salida legible para humanos                 | Muestra una salida legible por máquinas.                                                                 |

La asignación automática selecciona el primer puerto del registro que no esté en uso a partir de `19100`. Fleet rechaza los identificadores de inquilino duplicados y los puertos explícitos ya asignados a otra celda.

Las referencias de imagen se pasan como un único argumento al entorno de ejecución de contenedores. Se rechazan las referencias vacías y los valores que comiencen por `-` para evitar que una imagen se interprete como una opción de Docker o Podman.

El endpoint de Docker o Podman seleccionado debe ser local. Fleet rechaza los contextos remotos de Docker, los endpoints `DOCKER_HOST` y los servicios remotos de Podman antes de reservar un puerto o crear un estado local. No se admiten hosts de celdas remotos.

Cuando Fleet inicia una celda nueva, el proceso de creación espera hasta aproximadamente un minuto a que su Gateway responda a `/healthz`. Si la celda no pasa a estar en buen estado, Fleet conserva intactos su contenedor y su fila del registro para `fleet status`, `fleet logs` o la eliminación explícita. `--no-start` omite esta comprobación de estado. El token de Gateway generado para una celda nueva que no esté en buen estado no se pierde: permanece en el entorno del contenedor (`docker|podman inspect`) y, dado que la celda todavía no ha atendido tráfico, `fleet rm --force` seguido de una nueva creación siempre es una alternativa segura.

### Fijación mediante resumen

Los comandos de creación y actualización aceptan referencias de imagen fijadas mediante un resumen, como `--image ghcr.io/openclaw/openclaw@sha256:<digest>`. Fleet pasa la referencia de imagen literalmente a Docker o Podman, lo que permite mantener una celda en bytes de imagen inmutables en lugar de una etiqueta variable.

El resultado de la creación incluye el identificador del inquilino, el nombre del contenedor, el puerto del host, el token de Gateway y la URL local. Incluso en una salida JSON, trate el resultado como información secreta porque contiene el token.

### Límites de disco

`--disk` limita únicamente la capa escribible del contenedor. Los directorios de estado y autenticación por inquilino montados mediante enlace permanecen en el almacenamiento del host; use cuotas de proyecto del sistema de archivos del host cuando esos directorios también necesiten un límite estricto.

| Entorno de ejecución/backend de almacenamiento | Compatibilidad con `--disk`                                                             |
| ----------------------- | ---------------------------------------------------------------------------- |
| Docker overlay2 sobre XFS  | Requiere la opción de montaje de XFS `pquota`.                                      |
| Docker btrfs o zfs     | Compatible mediante el controlador de almacenamiento.                                             |
| Podman overlay          | Requiere almacenamiento subyacente XFS.                                                |
| Otros backends          | La creación del contenedor falla con el error del daemon y las indicaciones de Fleet sobre el backend. |

### Política de salida

| Modo       | Docker                                                                                                | Podman                                                                              |
| ---------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `bridge`   | Compatible; el tráfico saliente no está restringido de forma predeterminada.                                                | Compatible; el tráfico saliente no está restringido de forma predeterminada.                              |
| `internal` | Se rechaza porque Docker no conserva el puerto de Gateway de loopback publicado en una red interna. | Compatible; el Gateway de loopback permanece publicado mientras se bloquea el tráfico saliente. |

Para Docker, mantenga el modo puente y aplique la política de salida mediante reglas del firewall del host, como la cadena `DOCKER-USER`.

## `fleet list`

Enumere las celdas por orden de identificador de inquilino:

```bash
openclaw fleet list
openclaw fleet ls
openclaw fleet list --json
```

La tabla contiene:

| Columna    | Significado                                                                                                                                                                                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tenant`  | Identificador del inquilino.                                                                                                                                                                                                                                                                            |
| `state`   | Estado activo del contenedor obtenido mediante la inspección de Docker o Podman. `unknown` significa que el entorno de ejecución no estaba disponible o que existe un contenedor con el nombre de la celda, pero sus etiquetas de propiedad de Fleet no coinciden con el registro (una señal de colisión o manipulación; inspecciónelo manualmente antes de actuar). |
| `port`    | Puerto de host de loopback asignado al Gateway de la celda.                                                                                                                                                                                                                                        |
| `image`   | Imagen de contenedor registrada.                                                                                                                                                                                                                                                             |
| `created` | Hora de creación de la celda.                                                                                                                                                                                                                                                                   |

Las filas del registro permanecen visibles cuando Docker o Podman no están disponibles; solo el estado activo pasa a ser `unknown`.

## `fleet status`

Inspeccione una celda:

```bash
openclaw fleet status acme
openclaw fleet status acme --json
```

El estado combina la fila del registro de Fleet, la inspección del contenedor activo y una breve solicitud realizada con el máximo esfuerzo a:

```text
http://127.0.0.1:<host-port>/healthz
```

El resultado de la comprobación de estado es `ok`, `failed` o `skipped`. `/healthz` demuestra que el Gateway está activo, no que todos los canales o plugins configurados estén completamente listos. La comprobación se omite cuando no existe un endpoint local utilizable que comprobar.

## `fleet logs`

Transmita los registros del contenedor de una celda directamente al terminal:

```bash
openclaw fleet logs acme
openclaw fleet logs acme --follow
openclaw fleet logs acme --tail 200
openclaw fleet logs acme --since 10m
```

Fleet verifica las etiquetas de propiedad del contenedor registrado antes de leer cualquier registro, por lo que rechaza un contenedor ajeno que use el nombre de celda esperado. La transmisión queda fijada al identificador del contenedor inspeccionado, por lo que una sustitución simultánea no puede redirigirla a una generación más reciente. Presione Ctrl-C para finalizar `--follow` sin considerar la detención realizada por el operador como un error del comando. La salida de los registros pasa por un filtro de ocultación que sustituye el token de Gateway actual de la celda por `<redacted>` antes de que nada llegue al terminal.

`fleet logs` no tiene un modo `--json` porque los registros del contenedor son un flujo sin procesar de stdout/stderr. Para scripts, limite la salida con `--tail` y use redirecciones o pipelines normales del shell.

## `fleet start`, `fleet stop` y `fleet restart`

Controle una celda existente con su entorno de ejecución registrado:

```bash
openclaw fleet start acme
openclaw fleet stop acme
openclaw fleet restart acme
```

Estos comandos operan sobre el nombre de contenedor registrado. Fallan si el inquilino es desconocido o si el entorno de ejecución registrado no puede realizar la operación.

## `fleet upgrade`

Vuelva a descargar la imagen registrada y sustituya el contenedor de la celda:

```bash
openclaw fleet upgrade acme
```

Traslade la celda a otra imagen:

```bash
openclaw fleet upgrade acme --image ghcr.io/openclaw/openclaw:<version>
```

La actualización descarga la imagen de destino, inspecciona el contenedor existente y la red por celda, detiene y elimina el contenedor y, a continuación, lo vuelve a crear e iniciar. El contenedor sustituto conserva el mismo puerto del host, los directorios de datos, la red puente por celda, el perfil del entorno de ejecución, los límites de recursos, la política de reinicio, el entorno administrado por Fleet y los valores proporcionados originalmente con `--env`. El estado montado persiste tras sustituir el contenedor; el entorno predeterminado de la imagen puede cambiar con la imagen de destino.

La sustitución solo se confirma después de que su Gateway responda a `/healthz` en el puerto de bucle local de la celda, conforme al contrato de estado que utiliza el archivo de Compose oficial. Si el contenedor sustituto termina, entra en un bucle de fallos o no alcanza un estado correcto en aproximadamente un minuto, se elimina y se restaura el contenedor anterior, de modo que una imagen defectuosa no deje fuera de servicio una celda operativa.

El token del Gateway no se almacena intencionadamente en el registro de Fleet. Antes de eliminar el contenedor anterior, Fleet lee su entorno y transfiere `OPENCLAW_GATEWAY_TOKEN` al contenedor sustituto. No elimine manualmente el contenedor anterior antes de una actualización si el token no existe en ningún otro lugar bajo su control.

## `fleet backup` y `fleet restore`

Realice una copia de seguridad de una celda detenida:

```bash
openclaw fleet stop acme
openclaw fleet backup acme --out ./acme.tgz
```

Restaure ese archivo en la celda registrada:

```bash
openclaw fleet restore acme --from ./acme.tgz
```

Estos son comandos con privilegios de operador del host. Los archivos contienen el estado del inquilino y secretos de autenticación, se crean con el modo `0600` y deben almacenarse como credenciales. La copia de seguridad rechaza una celda en ejecución para que el estado de SQLite se capture de forma coherente. La restauración rechaza una celda en ejecución salvo que se proporcione `--force`, sustituye únicamente el estado de ese inquilino, renueva el token del Gateway e imprime el nuevo token una sola vez. Fleet realiza la copia de seguridad de un inquilino a la vez; la copia de seguridad de todos los inquilinos es una acción independiente del operador.

La restauración necesita un contenedor existente y detenido porque el perfil de entorno de ejecución obtenido mediante su inspección proporciona los límites, la asignación de usuario, la procedencia del entorno y la imagen del contenedor sustituto. Si el contenedor registrado se eliminó por otros medios, ejecute primero `fleet rm <tenant> --force` sin `--purge-data`, vuelva a crear la celda con la imagen prevista y `--no-start` y, a continuación, vuelva a intentar la restauración. La primera eliminación conserva intactos ambos directorios de datos del inquilino.

Ambos comandos aceptan `--max-bytes <bytes>` para limitar los datos de archivos archivados o extraídos, y ambos aplican el mismo presupuesto fijo de un millón de segmentos de ruta del archivo para que los archivos maliciosos compuestos únicamente por metadatos no puedan agotar los inodos del host y todas las copias de seguridad aceptadas sigan siendo restaurables. La copia de seguridad acepta `--out <path>` y ambos comandos admiten `--json`.

Los archivos solo contienen archivos normales y directorios. La copia de seguridad nunca sigue ni almacena enlaces simbólicos, enlaces físicos, sockets ni nodos de dispositivo; en el resultado se informa de las cantidades omitidas. La restauración rechaza los archivos que contengan cualquier otro tipo de entrada. Los árboles de enlaces simbólicos recreables, como `node_modules` del espacio de trabajo, deben volver a instalarse dentro de la celda después de una restauración.

## `fleet doctor`

Audite todas las celdas o un inquilino sin cambiar el estado del entorno de ejecución ni del sistema de archivos:

```bash
openclaw fleet doctor
openclaw fleet doctor acme --json
```

Doctor comprueba la localidad del entorno de ejecución, las etiquetas de propiedad, el estado, la protección, los límites de recursos, la vinculación del puerto de bucle local, la presencia del token, la propiedad de la red y el modo de salida, así como los permisos de los directorios de estado privados. Las advertencias describen las celdas detenidas o las diferencias de propiedad; cualquier comprobación fallida establece un código de salida del proceso distinto de cero.

## `fleet rm`

Elimine una celda detenida del entorno de ejecución y del registro, pero conserve los datos del inquilino:

```bash
openclaw fleet rm acme
```

Un contenedor en ejecución requiere `--force`:

```bash
openclaw fleet rm acme --force
```

Elimine también permanentemente los datos de la celda:

```bash
openclaw fleet rm acme --purge-data --force
```

Fleet elimina el contenedor de la celda antes de eliminar su red puente dedicada. `--purge-data` requiere `--force`. Antes de la eliminación recursiva, Fleet resuelve ambas raíces propiedad de Fleet y los dos directorios por inquilino. Cada destino debe ser exactamente la hoja de inquilino esperada, estar estrictamente dentro de su raíz y no ser un enlace simbólico. Estas comprobaciones de contención evitan que una ruta de registro dañada o un enlace simbólico entre inquilinos redirijan la eliminación a otra ubicación.

La purga puede volver a intentarse cuando un directorio de inquilino esperado exacto ya no existe. Esto permite que una invocación posterior complete la limpieza después de un fallo parcial del sistema de archivos sin relajar las comprobaciones de rutas para los directorios que aún existen.

## Diseño del almacenamiento y los contenedores

El estado de la celda y las claves de cifrado del perfil de autenticación utilizan rutas del host independientes por inquilino dentro del directorio de estado activo de OpenClaw:

```text
<state-dir>/fleet/cells/<tenant>/
<state-dir>/fleet/auth-profile-secrets/<tenant>/
```

El primer directorio se monta en `/home/node/.openclaw`. El segundo se monta en `/home/node/.config/openclaw`, en consonancia con el montaje de claves de cifrado de la configuración oficial de Docker. Por lo tanto, la clave de cifrado no queda expuesta bajo el montaje de estado ordinario ni se incluye cuando solo se realiza una copia de seguridad o se comparte el directorio de estado de la celda. Ambos directorios persisten tras una eliminación y actualización normales; `fleet rm --purge-data --force` elimina ambos después de realizar comprobaciones de contención independientes.

Antes del primer inicio, Fleet inicializa la configuración de la celda con `gateway.mode=local`, autenticación mediante token, la vinculación del contenedor a la LAN y los orígenes de Control UI para el puerto del host asignado. El valor del token no se escribe en esa configuración; permanece en el entorno del contenedor.

Fleet fija las rutas de contenedor de la imagen oficial con estos valores de entorno:

| Variable                 | Valor del contenedor                 |
| ------------------------ | ------------------------------------ |
| `HOME`                   | `/home/node`                         |
| `OPENCLAW_HOME`          | `/home/node`                         |
| `OPENCLAW_STATE_DIR`     | `/home/node/.openclaw`               |
| `OPENCLAW_CONFIG_PATH`   | `/home/node/.openclaw/openclaw.json` |
| `OPENCLAW_WORKSPACE_DIR` | `/home/node/.openclaw/workspace`     |
| `OPENCLAW_GATEWAY_TOKEN` | Token de celda generado o proporcionado |

La imagen oficial utiliza de forma predeterminada el usuario no root `node` con UID 1000. Fleet mantiene los montajes vinculados privados `0700` con permisos de escritura sin hacerlos accesibles para todos los usuarios. Docker con root ejecuta la celda con el UID y el GID del usuario no root que realiza la invocación; Docker sin root utiliza el UID 0 del contenedor, que se asigna al usuario sin privilegios del host que realiza la invocación dentro del espacio de nombres de usuario del daemon. Podman utiliza `keep-id` con el UID y el GID del usuario que realiza la invocación. Cuando Fleet se ejecuta como root con un entorno de ejecución con root, conserva el usuario de la imagen y asigna los archivos iniciales del montaje al UID/GID 1000.

En hosts con SELinux, los montajes de Docker y Podman reciben un reetiquetado privado `:Z`. Si restaura o traslada los datos de una celda, mantenga las rutas montadas mediante vinculación con permisos de escritura para el usuario efectivo del contenedor. El perfil es compatible con entornos sin root, pero Docker o Podman ya deben estar configurados para funcionar sin root en el host; Fleet no convierte un daemon con root en uno sin root.

## Perfil de seguridad

Fleet aplica el siguiente perfil a cada celda:

| Control                  | Perfil aplicado                                       | Motivo                                                                                          |
| ------------------------ | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Capacidades de Linux     | `--cap-drop=ALL`                                      | El Gateway es un proceso de Node.js y no necesita capacidades de Linux adicionales.             |
| Escalada de privilegios  | `--security-opt no-new-privileges`                    | Impide que los procesos obtengan privilegios mediante binarios setuid o setgid.                  |
| Proceso init             | `--init`                                              | Recolecta los procesos descendientes y reenvía las señales del ciclo de vida del contenedor.     |
| Límite de procesos       | `--pids-limit 512` de forma predeterminada            | Limita el agotamiento por bifurcaciones y procesos.                                              |
| Límite de memoria        | `--memory 2g` de forma predeterminada                 | Limita el uso de memoria de la celda.                                                            |
| Límite de CPU            | `--cpus 2` de forma predeterminada                    | Limita el uso de CPU de la celda.                                                                |
| Disco de capa escribible | `--disk` opcional                                     | Limita la capa del contenedor cuando el backend de almacenamiento admite cuotas.                 |
| Política de reinicio     | `--restart unless-stopped`                            | Reinicia una celda con errores sin anular una detención intencionada.                            |
| Publicación en el host   | Solo `127.0.0.1:<host-port>:18789`                    | Mantiene el Gateway fuera de las interfaces comodín del host.                                   |
| Red de la celda          | Una red puente o una red interna de Podman por celda   | Separa el tráfico por IP de los contenedores y, opcionalmente, bloquea la salida de Podman.       |
| Identidad del contenedor | Asignación de usuario coincidente con la del host      | Mantiene los montajes vinculados privados con permisos de escritura sin conceder acceso global.  |
| Estado persistente       | Montajes por celda; sin montaje de estado compartido   | Mantiene la configuración, las credenciales, las sesiones y los espacios de trabajo del inquilino en su árbol de datos. |
| Comando del contenedor   | `node dist/index.js gateway --bind lan --port 18789` | Escucha en la red del contenedor para que la asignación de puertos del host limitada al bucle local pueda alcanzarlo. |

Fleet nunca monta `/var/run/docker.sock`, utiliza `--privileged` ni redes del host, ni añade capacidades. El puente por celda es un límite de separación entre celdas, no un cortafuegos de salida: las celdas conservan la salida de red necesaria para los proveedores y canales. Anteponga al puerto de bucle local un proxy, un túnel SSH o una configuración de tailnet que se ajuste a su implementación. `http://127.0.0.1:<port>` solo es accesible directamente desde el host de Fleet.

Este perfil separa los contenedores de los inquilinos, pero no protege a los inquilinos del operador de Fleet, del administrador del entorno de ejecución de contenedores ni de un host comprometido. Consulte [Alojamiento multiinquilino](/es/gateway/multi-tenant-hosting) para conocer el modelo de confianza completo y las opciones de aislamiento más estrictas.

## Gestión de tokens

De forma predeterminada, `fleet create` genera un token de Gateway hexadecimal, criptográficamente aleatorio, de 32 caracteres y lo imprime una sola vez en el resultado de creación. Almacénelo en el gestor de secretos aprobado y evite capturar la salida de creación en los registros.

`--gateway-token` coloca un token personalizado en los argumentos del proceso local, que pueden conservarse en el historial del shell o ser visibles en las listas de procesos. Es preferible utilizar el token generado, salvo que un flujo de trabajo de gestión de secretos existente requiera un valor proporcionado.

El token y todos los valores pasados con `--env` se encuentran en el entorno del contenedor. Fleet los escribe en un archivo de entorno de corta duración con el modo `0600`, pasa únicamente la ruta de ese archivo a Docker o Podman y lo elimina cuando finaliza el comando del entorno de ejecución. Los valores escritos explícitamente en `openclaw fleet create --gateway-token ...` o `--env KEY=VALUE` aún pueden ser visibles en los argumentos del proceso externo `openclaw` y en el historial del shell.

Los valores del entorno del contenedor no están ocultos para el operador de confianza del host: los administradores de Docker o Podman pueden leerlos mediante la inspección del contenedor. La nota «se muestra una sola vez» de Fleet describe la salida normal de la CLI, no la resistencia frente a un administrador del host.

## Contenido relacionado

- [Alojamiento multiinquilino](/es/gateway/multi-tenant-hosting)
- [Docker](/es/install/docker)
- [Podman](/es/install/podman)
- [Seguridad del Gateway](/es/gateway/security)
