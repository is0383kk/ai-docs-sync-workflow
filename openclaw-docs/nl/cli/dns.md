---
read_when:
    - Je wilt wide-area discovery (DNS-SD) via Tailscale + CoreDNS
    - You're setting up split DNS for a custom discovery domain (example: openclaw.internal)
summary: CLI-referentie voor `openclaw dns` (helpers voor detectie via een groot netwerkgebied)
title: DNS
x-i18n:
    generated_at: "2026-07-27T05:46:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb07353df03f9d169e1aede2da0b711ffb68e8c9d21d51359e93e92cc0818ca2
    source_path: cli/dns.md
    workflow: 16
---

# `openclaw dns`

DNS-hulpprogramma's voor wide-area discovery (Tailscale + CoreDNS). Momenteel alleen macOS + Homebrew CoreDNS.

Gerelateerd:

- Gateway-discovery: [Discovery](/nl/gateway/discovery)
- Configuratie voor wide-area discovery: [Configuratie](/nl/gateway/configuration)

## `dns setup`

Plan of pas de CoreDNS-configuratie toe voor unicast DNS-SD-discovery.

```bash
openclaw dns setup
openclaw dns setup --domain openclaw.internal
openclaw dns setup --apply
```

| Optie              | Effect                                                                              |
| ------------------- | ----------------------------------------------------------------------------------- |
| `--domain <domain>` | Domein voor wide-area discovery (bijvoorbeeld `openclaw.internal`).                       |
| `--apply`           | Installeer/werk de CoreDNS-configuratie bij en (her)start de service. Vereist sudo, alleen macOS. |

Zonder `--domain` gebruikt OpenClaw `discovery.wideArea.domain` uit de configuratie.

Zonder `--apply` drukt de opdracht alleen het volgende af:

- Opgelost discovery-domein en pad naar zonebestand
- Huidige tailnet-IP-adressen
- Aanbevolen `openclaw.json`-discoveryconfiguratie
- Waarden voor Tailscale Split DNS-naamserver/domein om in te stellen in de Tailscale-beheerconsole

Met `--apply` (alleen macOS, vereist Homebrew CoreDNS):

- Initialiseert het zonebestand als dit ontbreekt
- Voegt de CoreDNS-importsectie toe als deze ontbreekt
- Herstart de `coredns`-brewservice

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Discovery](/nl/gateway/discovery)
