---
read_when:
    - Se está integrando el transporte sintético de control de calidad en una ejecución de pruebas local o de CI
    - Necesita la interfaz de configuración de qa-channel incluida.
    - Está iterando en la automatización de control de calidad de extremo a extremo
summary: Plugin de canal sintético tipo Slack para escenarios deterministas de control de calidad de OpenClaw
title: Canal de QA
x-i18n:
    generated_at: "2026-07-26T05:00:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a43c35e197116a6bd44b238010eb508aed23dea99ab872d10e6fc853b5f4d4a7
    source_path: channels/qa-channel.md
    workflow: 16
---

`qa-channel` es un transporte de mensajes sintético local al repositorio para el control de calidad automatizado de OpenClaw (`extensions/qa-channel`, paquete privado, excluido de las instalaciones empaquetadas). No es un canal de producción: existe para ejercitar el mismo límite de Plugin de canal que utilizan los transportes reales, a la vez que mantiene el estado determinista y totalmente inspeccionable.

## Qué hace

- Gramática de destinos de tipo Slack:
  - `dm:<user>`
  - `channel:<room>`
  - `group:<room>`
  - `thread:<room>/<thread>`
- Las conversaciones compartidas `channel:` y `group:` se presentan a los agentes como turnos de sala de grupo/canal, por lo que ejercitan la misma política de enrutamiento de respuestas visibles y herramientas de mensajes que utilizan Discord, Slack, Telegram y transportes similares.
- Bus sintético respaldado por HTTP para la inyección de mensajes entrantes, la captura de transcripciones salientes, la creación de hilos, las reacciones, las ediciones, las eliminaciones y las acciones de búsqueda/lectura.
- Ejecutor de autocomprobación del lado del host que escribe un informe Markdown en `.artifacts/qa-e2e/`.

## Configuración

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

Claves de cuenta:

- `enabled` - interruptor principal de esta cuenta.
- `name` - etiqueta de visualización opcional.
- `baseUrl` - URL del bus sintético. La cuenta se considera configurada una vez establecido este valor.
- `botUserId` - id. de usuario del bot sintético utilizado en la gramática de destinos (valor predeterminado: `openclaw`).
- `botDisplayName` - nombre para mostrar en los mensajes salientes (valor predeterminado: `OpenClaw QA`).
- `pollTimeoutMs` - intervalo de espera de sondeo largo. Entero entre 100 y 30000 (valor predeterminado: 1000).
- `allowFrom` - lista de remitentes permitidos (id. de usuario o `"*"`; valor predeterminado: `["*"]`). Los mensajes directos siempre aplican la política `open`; la política de grupos con lista de permitidos también utiliza estos id. de remitente sintéticos.
- `groupPolicy` - política de salas compartidas: `"open"` (valor predeterminado), `"allowlist"` o
  `"disabled"`.
- `groupAllowFrom` - lista opcional de remitentes permitidos para salas compartidas. Cuando se omite bajo
  `"allowlist"`, QA Channel recurre a `allowFrom`.
- `groups.<room>.requireMention` - exige una mención al bot antes de responder en una
  sala de grupo/canal específica (valor predeterminado: false). `groups."*"` establece el valor predeterminado;
  los valores `tools` / `toolsBySender` de cada sala establecen anulaciones de la política de herramientas.
- `defaultTo` - destino alternativo cuando no se proporciona ninguno.
- `actions.messages` / `actions.reactions` / `actions.search` / `actions.threads` - control de acceso a herramientas por acción.

Claves multicuenta del nivel superior:

- `accounts` - registro de anulaciones con nombre por cuenta, indexadas por id. de cuenta.
- `defaultAccount` - id. de cuenta preferida cuando hay varias configuradas.

## Ejecutores

Autocomprobación del lado del host (escribe un informe Markdown en `.artifacts/qa-e2e/`):

```bash
pnpm qa:e2e
```

Esto se enruta a través de `qa-lab`, inicia el bus de control de calidad del repositorio, arranca la sección del entorno de ejecución `qa-channel` y ejecuta una autocomprobación determinista.

Conjunto completo de escenarios respaldado por el repositorio:

```bash
pnpm openclaw qa suite
```

Ejecuta escenarios en paralelo en el carril del Gateway de control de calidad. Consulte la [descripción general del control de calidad](/es/concepts/qa-e2e-automation) para conocer los escenarios, los perfiles y los modos de proveedor.

Sitio de control de calidad respaldado por Docker (Gateway + interfaz de depuración de QA Lab en una sola pila):

```bash
pnpm qa:lab:up
```

Compila el sitio de control de calidad, inicia la pila de Gateway + QA Lab respaldada por Docker e imprime la URL de QA Lab. Desde allí se pueden seleccionar escenarios, elegir el carril del modelo, iniciar ejecuciones individuales y observar los resultados en directo. El depurador de QA Lab es independiente del paquete de Control UI distribuido.

## Relacionado

- [Descripción general del control de calidad](/es/concepts/qa-e2e-automation) - pila general, adaptadores de transporte, perfiles de Matrix y creación de escenarios
- [Emparejamiento](/es/channels/pairing)
- [Grupos](/es/channels/groups)
- [Descripción general de los canales](/es/channels)
