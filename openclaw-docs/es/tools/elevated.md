---
read_when:
    - Ajuste de los valores predeterminados del modo elevado, las listas de permitidos o el comportamiento de los comandos de barra diagonal
    - Cómo pueden acceder al host los agentes aislados en un entorno restringido
summary: 'Modo de ejecución elevada: ejecutar comandos fuera del entorno aislado desde un agente aislado'
title: Modo elevado
x-i18n:
    generated_at: "2026-07-26T05:31:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40627217acf56122acfc48b689be1b9e2c61d889fe698e9c3c8fd91270d4a6cf
    source_path: tools/elevated.md
    workflow: 16
---

Cuando un agente se ejecuta dentro de un entorno aislado, sus comandos `exec` quedan confinados al entorno aislado. El **modo elevado** permite que el agente salga de él y ejecute comandos fuera del entorno aislado, con controles de aprobación configurables.

<Info>
  El modo elevado solo cambia el comportamiento cuando el agente está **aislado**. Para los agentes no aislados, exec ya se ejecuta en el host.
</Info>

## Directivas

Controle el modo elevado por sesión con comandos de barra:

| Directiva        | Qué hace                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `/elevated on`   | Ejecuta fuera del entorno aislado en la ruta del host configurada y mantiene las aprobaciones                                                             |
| `/elevated ask`  | Igual que `on` (alias)                                                                                                            |
| `/elevated full` | Ejecuta fuera del entorno aislado en la ruta del host configurada y omite las aprobaciones cuando la política de aprobación del modo/host ya es permisiva |
| `/elevated off`  | Vuelve a la ejecución confinada al entorno aislado                                                                                            |

También está disponible como `/elev on|off|ask|full`.

Envíe `/elevated` sin argumentos para ver el nivel actual.

## Cómo funciona

<Steps>
  <Step title="Comprobar la disponibilidad">
    El modo elevado debe estar habilitado en la configuración y el remitente debe estar en la lista de permitidos:

    ```json5
    {
      tools: {
        elevated: {
          enabled: true,
          allowFrom: {
            discord: ["user-id-123"],
            whatsapp: ["+15555550123"],
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Establecer el nivel">
    Envíe un mensaje que contenga únicamente la directiva para establecer el valor predeterminado de la sesión:

    ```
    /elevated full
    ```

    O utilícela en línea (se aplica únicamente a ese mensaje):

    ```
    /elevated on ejecuta el script de despliegue
    ```

  </Step>

  <Step title="Los comandos se ejecutan fuera del entorno aislado">
    Con el modo elevado activo, las llamadas a `exec` salen del entorno aislado. El host efectivo es
    `gateway` de forma predeterminada, o `node` cuando el destino de exec configurado o de la sesión es
    `node`. En el modo `full`, las aprobaciones de exec se omiten cuando la política de aprobación
    del modo/host de exec resuelta ya es totalmente permisiva (seguridad `full`,
    solicitud `off`); de lo contrario, se sigue aplicando la política de aprobación normal. En el
    modo `on`/`ask`, siempre se aplican las reglas de aprobación configuradas.
  </Step>
</Steps>

## Orden de resolución

1. **Directiva en línea** en el mensaje (se aplica únicamente a ese mensaje)
2. **Anulación de sesión** (se establece enviando un mensaje que contenga únicamente una directiva)
3. **Valor predeterminado global** (`agents.defaults.elevatedDefault` en la configuración)

## Disponibilidad y listas de permitidos

- **Control global**: `tools.elevated.enabled` (debe ser `true`)
- **Lista de remitentes permitidos**: `tools.elevated.allowFrom` con listas por canal
- **Control por agente**: `agents.entries.*.tools.elevated.enabled` (solo puede imponer más restricciones; tanto el control global como el control por agente deben ser `true`)
- **Lista de permitidos por agente**: `agents.entries.*.tools.elevated.allowFrom` (el remitente debe coincidir tanto con la lista global como con la lista por agente)
- **Lista de permitidos alternativa proporcionada por el canal**: los plugins de canal pueden proporcionar opcionalmente una lista de permitidos alternativa mediante un hook adaptador del SDK, que se utiliza cuando `tools.elevated.allowFrom.<provider>` no está configurado. Actualmente, ningún canal incluido implementa este hook, por lo que, en la práctica, todos los proveedores necesitan hoy una entrada `tools.elevated.allowFrom.<provider>` explícita.
- **Deben superarse todos los controles**; de lo contrario, el modo elevado se considera no disponible

Formatos de las entradas de la lista de permitidos:

| Prefijo                  | Coincide con                         |
| ----------------------- | ------------------------------- |
| (ninguno)                  | ID del remitente, E.164 o campo From |
| `name:`                 | Nombre para mostrar del remitente             |
| `username:`             | Nombre de usuario del remitente                 |
| `tag:`                  | Etiqueta del remitente                      |
| `id:`, `from:`, `e164:` | Selección explícita de identidad     |

## Qué no controla el modo elevado

- **Política de herramientas**: si la política de herramientas deniega `exec`, el modo elevado no puede anularla.
- **Política de selección de host**: el modo elevado no convierte `auto` en una anulación libre entre hosts. Utiliza las reglas del destino de exec configurado o de la sesión y solo elige `node` cuando el destino ya es `node`.
- **Independiente de `/exec`**: la directiva `/exec` ajusta los valores predeterminados de exec por sesión (host, seguridad, solicitud, nodo) para los remitentes autorizados y no requiere el modo elevado.

<Note>
  El comando de chat de bash (prefijo `!`; alias `/bash`) es un control independiente que requiere que `tools.elevated` esté habilitado además de su propia opción `tools.bash.enabled`. Deshabilitar el modo elevado también bloquea los comandos de shell `!`.
</Note>

## Relacionado

<CardGroup cols={2}>
  <Card title="Herramienta exec" href="/es/tools/exec" icon="terminal">
    Ejecución de comandos de shell desde el agente.
  </Card>
  <Card title="Aprobaciones de exec" href="/es/tools/exec-approvals" icon="shield">
    Sistema de aprobación y listas de permitidos para `exec`.
  </Card>
  <Card title="Aislamiento" href="/es/gateway/sandboxing" icon="box">
    Configuración del entorno aislado a nivel del Gateway.
  </Card>
  <Card title="Entorno aislado frente a política de herramientas frente a modo elevado" href="/es/gateway/sandbox-vs-tool-policy-vs-elevated" icon="scale-balanced">
    Cómo se combinan los tres controles durante una llamada a una herramienta.
  </Card>
</CardGroup>
