---
read_when:
    - Mantiene scripts antiguos mediante `openclaw clawbot ...`
    - Se necesita orientación para migrar a los comandos actuales
summary: Referencia de la CLI para `openclaw clawbot` (espacio de nombres de alias heredado)
title: Clawbot
x-i18n:
    generated_at: "2026-07-26T04:33:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6baf9b4e9bbe8bb31cdc4923c38cd45a883b6e5be921a403335e257dacdc2cd5
    source_path: cli/clawbot.md
    workflow: 16
---

# `openclaw clawbot`

Espacio de nombres de alias heredado que se mantiene para la compatibilidad con versiones anteriores. Registra el mismo comando QR que la CLI de nivel superior, por lo que `openclaw clawbot qr` acepta todas las opciones de [`openclaw qr`](/es/cli/qr).

## Migración

Se recomienda usar el comando moderno de nivel superior:

- `openclaw clawbot qr` -> `openclaw qr`

## Relacionado

- [Referencia de la CLI](/es/cli)
