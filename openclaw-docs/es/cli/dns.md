---
read_when:
    - Se desea el descubrimiento de área amplia (DNS-SD) mediante Tailscale + CoreDNS
    - You're setting up split DNS for a custom discovery domain (example: openclaw.internal)
summary: Referencia de la CLI para `openclaw dns` (herramientas auxiliares de descubrimiento de área amplia)
title: DNS
x-i18n:
    generated_at: "2026-07-26T05:08:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb07353df03f9d169e1aede2da0b711ffb68e8c9d21d51359e93e92cc0818ca2
    source_path: cli/dns.md
    workflow: 16
---

# `openclaw dns`

Herramientas auxiliares de DNS para el descubrimiento de área amplia (Tailscale + CoreDNS). Actualmente, solo macOS + CoreDNS de Homebrew.

Relacionado:

- Descubrimiento del Gateway: [Descubrimiento](/es/gateway/discovery)
- Configuración del descubrimiento de área amplia: [Configuración](/es/gateway/configuration)

## `dns setup`

Planifica o aplica la configuración de CoreDNS para el descubrimiento DNS-SD mediante unidifusión.

```bash
openclaw dns setup
openclaw dns setup --domain openclaw.internal
openclaw dns setup --apply
```

| Opción              | Efecto                                                                              |
| ------------------- | ----------------------------------------------------------------------------------- |
| `--domain <domain>` | Dominio de descubrimiento de área amplia (por ejemplo, `openclaw.internal`).                       |
| `--apply`           | Instala o actualiza la configuración de CoreDNS y (re)inicia el servicio. Requiere sudo; solo para macOS. |

Sin `--domain`, OpenClaw utiliza `discovery.wideArea.domain` de la configuración.

Sin `--apply`, el comando solo muestra:

- El dominio de descubrimiento resuelto y la ruta del archivo de zona
- Las direcciones IP actuales de la tailnet
- La configuración de descubrimiento recomendada de `openclaw.json`
- Los valores del servidor de nombres y del dominio de Split DNS de Tailscale que se deben establecer en la consola de administración de Tailscale

Con `--apply` (solo para macOS; requiere CoreDNS de Homebrew):

- Inicializa el archivo de zona si no existe
- Añade la directiva de importación de CoreDNS si no existe
- Reinicia el servicio de brew `coredns`

## Relacionado

- [Referencia de la CLI](/es/cli)
- [Descubrimiento](/es/gateway/discovery)
