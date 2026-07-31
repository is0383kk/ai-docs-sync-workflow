---
read_when:
    - Ejecución del control de calidad de escritorio de Mantis Slack desde GitHub o localmente
    - Depuración de ejecuciones lentas de Mantis en la aplicación de escritorio de Slack
    - Elegir entre el modo de origen, prehidratado o de arrendamiento activo
    - Publicar capturas de pantalla y vídeos como evidencia en un PR
summary: 'Manual de operaciones para el control de calidad de Mantis en la aplicación de escritorio de Slack: ejecución desde GitHub, CLI local, sesiones VNC precalentadas, modos de hidratación, interpretación de tiempos, artefactos y gestión de errores.'
title: Manual de operaciones de Mantis para la aplicación de escritorio de Slack
x-i18n:
    generated_at: "2026-07-26T05:10:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack desktop QA es la vía de interfaz real para errores de tipo Slack que necesitan un
escritorio Linux, recuperación mediante VNC, Slack Web, un Gateway real de OpenClaw, capturas de pantalla,
vídeos y un comentario de evidencia en el PR. Úsela cuando las pruebas unitarias o la vía en vivo de Slack
sin interfaz gráfica no puedan demostrar el error.

## Modelo de almacenamiento

Mantis utiliza tres capas de almacenamiento:

- **Imagen del proveedor** - propiedad de Crabbox, almacenada en la cuenta del proveedor de nube.
  Contiene las capacidades de la máquina (Chrome/Chromium, ffmpeg, scrot,
  Node/corepack/pnpm, herramientas de compilación nativas) y directorios de caché vacíos.
- **Estado del arrendamiento activo** - propiedad de la sesión actual del operador. Puede contener un
  perfil de navegador con sesión iniciada, `/var/cache/crabbox/pnpm` y un checkout del código fuente
  preparado mientras el arrendamiento esté activo.
- **Artefactos de Mantis** - propiedad de la ejecución de OpenClaw. Se encuentran en
  `.artifacts/qa-e2e/mantis/...`; GitHub Actions los carga y la aplicación de GitHub de Mantis
  publica comentarios con evidencia en línea en el PR.

Nunca incluya secretos, cookies del navegador, el estado de inicio de sesión de Slack, checkouts del repositorio,
`node_modules` ni `dist/` en una imagen del proveedor.

## Despacho de GitHub

Ejecute el flujo de trabajo desde `main`:

```bash
gh workflow run mantis-slack-desktop-smoke.yml \
  --ref main \
  -f candidate_ref=<trusted-ref-or-sha> \
  -f pr_number=<pr-number> \
  -f scenario_id=slack-canary \
  -f crabbox_provider=aws \
  -f keep_vm=false \
  -f hydrate_mode=source
```

`candidate_ref` está restringido porque el flujo de trabajo utiliza credenciales reales: debe
resolverse como parte de la ascendencia actual de `main`, una etiqueta de versión o la cabecera de un PR abierto en
`openclaw/openclaw`.

El flujo de trabajo produce:

- artefacto cargado `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- comentario en línea en el PR de la aplicación de GitHub de Mantis
- `slack-desktop-smoke.png`, `slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`, `slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`, `mantis-slack-desktop-smoke-report.md`
- registros remotos: `slack-desktop-command.log`, `openclaw-gateway.log`, `chrome.log`, `ffmpeg.log`

El comentario del PR se actualiza en el mismo lugar mediante el marcador oculto `<!-- mantis-slack-desktop-smoke -->`.

## CLI local

Prueba en frío desde el código fuente:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --credential-source convex \
  --credential-role maintainer \
  --provider-mode live-frontier \
  --model openai/gpt-5.4 \
  --alt-model openai/gpt-5.4 \
  --scenario slack-canary \
  --hydrate-mode source
```

Mantenga la máquina virtual para la recuperación mediante VNC:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Abra VNC:

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

Reutilice un arrendamiento activo:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

Utilice `--hydrate-mode prehydrated` solo cuando el espacio de trabajo remoto reutilizado ya
tenga `node_modules` y un `dist/` compilado; de lo contrario, Mantis se cierra de forma segura.

Demuestre la interfaz nativa de aprobación de Slack:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` es mutuamente excluyente con `--gateway-setup`. Ejecuta
los escenarios opcionales `slack-approval-exec-native` y `slack-approval-plugin-native`,
a menos que se proporcione un `--scenario` explícito de punto de control de aprobación; los demás
escenarios de Slack se rechazan antes de que se inicie la máquina virtual. El ejecutor de QA de Slack escribe
cada archivo JSON de punto de control a partir del mensaje real de la API de Slack que observó y, a continuación,
el observador remoto representa ese mensaje en
`approval-checkpoints/<scenario>-pending.png` y
`approval-checkpoints/<scenario>-resolved.png`. La ejecución falla si falta o está vacío algún
JSON de punto de control, evidencia del mensaje, JSON de confirmación o captura de pantalla representada.

Los arrendamientos en frío de GitHub Actions no contienen cookies de Slack Web, por lo que la captura del navegador
puede mostrar la pantalla de inicio de sesión de Slack. Para la prueba de puntos de control de aprobación, confíe en las
imágenes representadas de los puntos de control y en los artefactos de QA de Slack en lugar de
`slack-desktop-smoke.png`. Utilice únicamente un arrendamiento activo conservado con un perfil
de Slack Web en el que se haya iniciado sesión manualmente cuando la propia captura de pantalla del navegador deba mostrar
Slack Web.

## Modos de hidratación

| Modo          | Cuándo usarlo                                  | Comportamiento remoto                                                                       | Contrapartida                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | Prueba normal de PR, máquinas en frío, CI        | Ejecuta `pnpm install --frozen-lockfile --prefer-offline` y `pnpm build` dentro de la máquina virtual | El más lento, pero ofrece la prueba más sólida desde el checkout del código fuente                 |
| `prehydrated` | Se preparó intencionadamente un arrendamiento reutilizado | Requiere `node_modules` y `dist/` existentes; omite la instalación y la compilación                     | Rápido, pero solo válido para arrendamientos activos controlados por el operador |

GitHub Actions siempre prepara el checkout candidato antes de la ejecución de la máquina virtual. Su
almacén de pnpm se almacena en caché según el sistema operativo, la versión de Node y el archivo de bloqueo. La ejecución de `source` en la máquina virtual
también reutiliza `/var/cache/crabbox/pnpm` cuando está presente.

## Interpretación de los tiempos

`mantis-slack-desktop-smoke-report.md` incluye los tiempos de las fases:

- `crabbox.warmup` - arranque del proveedor de nube, disponibilidad del escritorio/navegador y SSH.
- `crabbox.inspect` - consulta de los metadatos del arrendamiento.
- `credentials.prepare` - adquisición del arrendamiento de credenciales de Convex.
- `crabbox.remote_run` - sincronización, inicio del navegador, instalación/compilación de OpenClaw o
  validación de la hidratación, inicio del Gateway, captura de pantalla y grabación de vídeo.
- `artifacts.copy` - rsync de vuelta desde la máquina virtual.

`crabbox.remote_run` puede mostrar `accepted` cuando Crabbox devuelve un estado remoto distinto de cero,
pero Mantis copió metadatos que demuestran que se completó la configuración del Gateway de OpenClaw
o que el propio comando de QA de Slack terminó correctamente. Considere
`accepted` como una ejecución aprobada con explicación, no como un escenario fallido.

Si una ejecución es lenta:

- Predomina el calentamiento: precompile o promueva una imagen mejor del proveedor de Crabbox.
- Predomina `remote_run` en `source`: utilice un arrendamiento activo, mejore la reutilización
  del almacén de pnpm o traslade los prerrequisitos de la máquina a la imagen del proveedor.
- Predomina `remote_run` en `prehydrated`: el espacio de trabajo remoto no estaba
  realmente preparado, o la configuración del Gateway, el navegador o Slack es lenta.
- Predomina la copia de artefactos: examine el tamaño del vídeo y el contenido del directorio de artefactos.

## Lista de comprobación de evidencias

Un buen comentario de PR muestra:

- identificador del escenario y SHA candidato
- URL de la ejecución de GitHub Actions y URL del artefacto
- captura de pantalla en línea del punto de control de aprobación o una captura de Slack Web desde un
  arrendamiento activo con sesión iniciada
- vista previa animada en línea cuando esté disponible
- enlaces al MP4 completo y al MP4 recortado
- estado de aprobación/error y resumen de tiempos del informe

No confirme capturas de pantalla ni vídeos en el repositorio. Manténgalos en los artefactos de GitHub
Actions o en el comentario del PR.

## Gestión de errores

Si el flujo de trabajo falla antes de la ejecución de la máquina virtual, examine primero el trabajo de Actions.
Las causas habituales son un `candidate_ref` no confiable, secretos de entorno ausentes o un
error de instalación/compilación del candidato.

Si la ejecución de la máquina virtual falla pero se copiaron las capturas de pantalla, examine:

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

Si la ejecución conservó el arrendamiento, abra VNC con el comando `crabbox vnc ...`
del informe y, a continuación, detenga el arrendamiento cuando termine:

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

Si el inicio de sesión de Slack ha caducado, repárelo mediante VNC en un arrendamiento conservado y vuelva a ejecutar con
`--lease-id`. No incluya ese perfil del navegador en una imagen del proveedor.

## Temas relacionados

- [Descripción general de QA](/es/concepts/qa-e2e-automation)
- [Canal de Slack](/es/channels/slack)
- [Pruebas](/es/help/testing)
