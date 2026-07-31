---
read_when: You want per-agent sandboxing or per-agent tool allow/deny policies in a multi-agent gateway.
sidebarTitle: Multi-agent sandbox and tools
status: active
summary: Sandbox por agente + restricciones de herramientas, precedencia y ejemplos
title: Sandbox y herramientas multiagente
x-i18n:
    generated_at: "2026-07-26T04:55:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0e07d07c30b844be1e1d93db62fcdaab72c47a5248367559642a959bf09ad193
    source_path: tools/multi-agent-sandbox-tools.md
    workflow: 16
---

Cada agente de una configuración multiagente puede reemplazar la política global de aislamiento y herramientas. Esta página abarca la configuración por agente, las reglas de precedencia y ejemplos.

<CardGroup cols={3}>
  <Card title="Aislamiento" href="/es/gateway/sandboxing">
    Backends y modos: referencia completa del aislamiento.
  </Card>
  <Card title="Aislamiento frente a política de herramientas frente a modo elevado" href="/es/gateway/sandbox-vs-tool-policy-vs-elevated">
    Depure «¿por qué está bloqueado?»
  </Card>
  <Card title="Modo elevado" href="/es/tools/elevated">
    Ejecución elevada para remitentes de confianza.
  </Card>
</CardGroup>

<Warning>
La autenticación se limita por agente: cada agente tiene su propio almacén de autenticación `agentDir` en `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. Nunca reutilice `agentDir` entre agentes. Los agentes pueden consultar los perfiles de autenticación del agente predeterminado/principal cuando no tienen un perfil local, pero los tokens de actualización de OAuth no se clonan en los almacenes de agentes secundarios. Si copia las credenciales manualmente, copie únicamente perfiles estáticos portátiles `api_key` o `token`.
</Warning>

---

## Ejemplos de configuración

<AccordionGroup>
  <Accordion title="Ejemplo 1: Agente personal + agente familiar restringido">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "name": "Personal Assistant",
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "family",
            "name": "Family Bot",
            "workspace": "~/.openclaw/workspace-family",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read", "message"],
              "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"],
              "message": {
                "crossContext": {
                  "allowWithinProvider": false,
                  "allowAcrossProviders": false
                }
              }
            }
          }
        ]
      },
      "bindings": [
        {
          "agentId": "family",
          "match": {
            "provider": "whatsapp",
            "accountId": "*",
            "peer": {
              "kind": "group",
              "id": "120363424282127706@g.us"
            }
          }
        }
      ]
    }
    ```

    **Resultado:**

    - Agente `main`: se ejecuta en el host, con acceso completo a las herramientas.
    - Agente `family`: se ejecuta en Docker (un contenedor por agente), solo `read` y envíos de mensajes en la conversación actual.

  </Accordion>
  <Accordion title="Ejemplo 2: Agente de trabajo con aislamiento compartido">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "personal",
            "workspace": "~/.openclaw/workspace-personal",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "work",
            "workspace": "~/.openclaw/workspace-work",
            "sandbox": {
              "mode": "all",
              "scope": "shared",
              "workspaceRoot": "/tmp/work-sandboxes"
            },
            "tools": {
              "allow": ["read", "write", "apply_patch", "exec"],
              "deny": ["browser", "gateway", "discord"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
  <Accordion title="Ejemplo 2b: Perfil global de programación + agente solo de mensajería">
    ```json
    {
      "tools": { "profile": "coding" },
      "agents": {
        "list": [
          {
            "id": "support",
            "tools": { "profile": "messaging", "allow": ["slack"] }
          }
        ]
      }
    }
    ```

    **Resultado:**

    - Los agentes predeterminados obtienen herramientas de programación.
    - El agente `support` es solo de mensajería (+ herramienta de Slack).

  </Accordion>
  <Accordion title="Ejemplo 3: Distintos modos de aislamiento por agente">
    ```json
    {
      "agents": {
        "defaults": {
          "sandbox": {
            "mode": "non-main",
            "scope": "session"
          }
        },
        "list": [
          {
            "id": "main",
            "workspace": "~/.openclaw/workspace",
            "sandbox": {
              "mode": "off"
            }
          },
          {
            "id": "public",
            "workspace": "~/.openclaw/workspace-public",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read"],
              "deny": ["exec", "write", "edit", "apply_patch"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
</AccordionGroup>

---

## Precedencia de la configuración

Cuando existen tanto configuraciones globales (`agents.defaults.*`) como específicas del agente (`agents.entries.*.*`):

### Configuración del aislamiento

La configuración específica del agente reemplaza la global:

```text
agents.entries.*.sandbox.mode > agents.defaults.sandbox.mode
agents.entries.*.sandbox.scope > agents.defaults.sandbox.scope
agents.entries.*.sandbox.workspaceRoot > agents.defaults.sandbox.workspaceRoot
agents.entries.*.sandbox.workspaceAccess > agents.defaults.sandbox.workspaceAccess
agents.entries.*.sandbox.docker.* > agents.defaults.sandbox.docker.*
agents.entries.*.sandbox.browser.* > agents.defaults.sandbox.browser.*
agents.entries.*.sandbox.prune.* > agents.defaults.sandbox.prune.*
```

<Note>
`agents.entries.*.sandbox.{docker,browser,prune}.*` reemplaza `agents.defaults.sandbox.{docker,browser,prune}.*` para ese agente (se ignora cuando el ámbito del aislamiento se resuelve como `"shared"`).
</Note>

### Restricciones de herramientas

El orden de filtrado es:

<Steps>
  <Step title="Perfil de herramientas">
    `tools.profile` o `agents.entries.*.tools.profile`.
  </Step>
  <Step title="Perfil de herramientas del proveedor">
    `tools.byProvider[provider].profile` o `agents.entries.*.tools.byProvider[provider].profile`.
  </Step>
  <Step title="Política global de herramientas">
    `tools.allow` / `tools.deny`.
  </Step>
  <Step title="Política de herramientas del proveedor">
    `tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="Política de herramientas específica del agente">
    `agents.entries.*.tools.allow/deny`.
  </Step>
  <Step title="Política del proveedor del agente">
    `agents.entries.*.tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="Política de herramientas del aislamiento">
    `tools.sandbox.tools` o `agents.entries.*.tools.sandbox.tools`.
  </Step>
  <Step title="Política de herramientas del subagente">
    `tools.subagents.tools`, si corresponde.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Reglas de precedencia">
    - Cada nivel puede restringir aún más las herramientas, pero no puede volver a conceder herramientas denegadas en niveles anteriores.
    - Si se establece `agents.entries.*.tools.sandbox.tools`, reemplaza `tools.sandbox.tools` para ese agente.
    - Si se establece `agents.entries.*.tools.profile`, reemplaza `tools.profile` para ese agente.
    - Las claves de herramientas del proveedor aceptan `provider` (por ejemplo, `google-antigravity`) o `provider/model` (por ejemplo, `openai/gpt-5.4`).

  </Accordion>
  <Accordion title="Comportamiento de una lista de permitidos vacía">
    Si alguna lista de permitidos explícita de esa cadena deja la ejecución sin herramientas invocables, OpenClaw se detiene antes de enviar el prompt al modelo. Esto es intencional: un agente configurado con una herramienta ausente como `agents.entries.*.tools.allow: ["query_db"]` debe fallar de forma explícita hasta que se habilite el plugin que registra `query_db`, en lugar de continuar como agente solo de texto.
  </Accordion>
</AccordionGroup>

Las políticas de herramientas admiten formas abreviadas `group:*` que se expanden a varias herramientas. Consulte [Grupos de herramientas](/es/gateway/sandbox-vs-tool-policy-vs-elevated#tool-groups-shorthands) para ver la lista completa.

Las sustituciones elevadas por agente (`agents.entries.*.tools.elevated`) pueden restringir aún más la ejecución elevada para agentes específicos. Consulte [Modo elevado](/es/tools/elevated) para obtener más información.

---

## Migración desde un único agente

<Tabs>
  <Tab title="Antes (un solo agente)">
    ```json
    {
      "agents": {
        "defaults": {
          "workspace": "~/.openclaw/workspace",
          "sandbox": {
            "mode": "non-main"
          }
        }
      },
      "tools": {
        "sandbox": {
          "tools": {
            "allow": ["read", "write", "apply_patch", "exec"],
            "deny": []
          }
        }
      }
    }
    ```
  </Tab>
  <Tab title="Después (multiagente)">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          }
        ]
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
Las claves de configuración heredadas `agents.defaults.*`/`agents.entries.*.*` (como `sandbox.perSession`, `agentRuntime`, `embeddedPi`) se migran mediante `openclaw doctor`; de ahora en adelante, se recomienda usar `agents.defaults` + `agents.entries`.
</Note>

---

## Ejemplos de restricciones de herramientas

<Tabs>
  <Tab title="Agente de solo lectura">
    ```json
    {
      "tools": {
        "allow": ["read"],
        "deny": ["exec", "write", "edit", "apply_patch", "process"]
      }
    }
    ```
  </Tab>
  <Tab title="Ejecución de shell con las herramientas del sistema de archivos deshabilitadas">
    ```json
    {
      "tools": {
        "allow": ["read", "exec", "process"],
        "deny": ["write", "edit", "apply_patch", "browser", "gateway"]
      }
    }
    ```

    <Warning>
    Esta política deshabilita las herramientas del sistema de archivos de OpenClaw, pero `exec` sigue siendo un shell y puede escribir archivos en cualquier lugar que permita el sistema de archivos del host o aislamiento seleccionado. Para un agente de solo lectura, deniegue `exec` y `process`, o combine el acceso al shell con controles del sistema de archivos del aislamiento, como `agents.defaults.sandbox.workspaceAccess: "ro"` o `"none"`.
    </Warning>

  </Tab>
  <Tab title="Solo comunicación">
    ```json
    {
      "tools": {
        "sessions": { "visibility": "tree" },
        "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
        "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
      }
    }
    ```

    `sessions_history` en este perfil sigue devolviendo una vista de recuperación limitada y saneada en lugar de un volcado de la transcripción sin procesar. La recuperación del asistente elimina etiquetas de razonamiento, estructuras auxiliares `<relevant-memories>`, cargas XML de llamadas a herramientas en texto sin formato (incluidos `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` y bloques truncados de llamadas a herramientas), estructuras auxiliares degradadas de llamadas a herramientas, tokens de control del modelo ASCII/de ancho completo filtrados y XML malformado de llamadas a herramientas de MiniMax antes de la ocultación/truncamiento.

  </Tab>
</Tabs>

---

## Error común: "non-main"

<Warning>
`agents.defaults.sandbox.mode: "non-main"` comprueba la clave de sesión con respecto a la clave de la sesión principal (siempre `"main"`; `session.mainKey` no es configurable por el usuario, y OpenClaw advierte e ignora cualquier otro valor), no el identificador del agente. Las sesiones de grupo/canal siempre obtienen sus propias claves, por lo que se tratan como no principales y se aislarán. Si desea que un agente nunca se aísle, establezca `agents.entries.*.sandbox.mode: "off"`.
</Warning>

---

## Pruebas

Después de configurar el aislamiento y las herramientas multiagente:

<Steps>
  <Step title="Comprobar la resolución de agentes">
    ```bash
    openclaw agents list --bindings
    ```
  </Step>
  <Step title="Verificar los contenedores de aislamiento">
    ```bash
    docker ps --filter "name=openclaw-sbx-"
    ```
  </Step>
  <Step title="Probar las restricciones de herramientas">
    - Envíe un mensaje que requiera herramientas restringidas.
    - Compruebe que el agente no pueda usar las herramientas denegadas.

  </Step>
  <Step title="Supervisar los registros">
    ```bash
    openclaw logs --follow | grep -E "routing|sandbox|tools"
    ```
  </Step>
</Steps>

---

## Solución de problemas

<AccordionGroup>
  <Accordion title="El agente no está aislado a pesar de `mode: 'all'`">
    - Compruebe si existe un `agents.defaults.sandbox.mode` global que lo reemplace.
    - La configuración específica del agente tiene precedencia, por lo que debe establecer `agents.entries.*.sandbox.mode: "all"`.

  </Accordion>
  <Accordion title="Herramientas aún disponibles pese a la lista de denegación">
    - Consulte el [orden de filtrado completo](#tool-restrictions): perfil → perfil del proveedor → política global → política del proveedor → política del agente → política del proveedor del agente → entorno aislado → subagente.
    - Cada nivel solo puede imponer más restricciones, no volver a conceder permisos.
    - Consulte [Entorno aislado frente a política de herramientas frente a modo elevado](/es/gateway/sandbox-vs-tool-policy-vs-elevated) para depurar paso a paso.

  </Accordion>
  <Accordion title="El contenedor no está aislado por agente">
    - El valor predeterminado de `scope` es `"agent"` (un contenedor por id. de agente).
    - Establezca `scope: "session"` para usar un contenedor por sesión, o `scope: "shared"` para reutilizar un contenedor entre agentes.

  </Accordion>
</AccordionGroup>

---

## Temas relacionados

- [Modo elevado](/es/tools/elevated)
- [Enrutamiento multiagente](/es/concepts/multi-agent)
- [Configuración del entorno aislado](/es/gateway/config-agents#agentsdefaultssandbox)
- [Entorno aislado frente a política de herramientas frente a modo elevado](/es/gateway/sandbox-vs-tool-policy-vs-elevated) — depuración de «¿por qué está bloqueado?»
- [Aislamiento](/es/gateway/sandboxing) — referencia completa del entorno aislado (modos, ámbitos, backends e imágenes)
- [Gestión de sesiones](/es/concepts/session)
